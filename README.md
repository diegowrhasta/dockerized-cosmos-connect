# Dockerized cosmos connect

This is a small prototype to test out getting CosmosDB emulator up and running on a
Docker container. And being able to connect to it normally through _https_. **And**
having the service that connects to it being also dockerized.

## The Problem

Because the cosmos emulator always tries to use SSL, if we try to connect to it
directly due to its _testing nature_ it will hold a dev-certificate that will fail
the handshake since it won't be a trusted certificate. And so we have to trust the
certificate. A good place to start is to understand what's the CosmosDB Emulator:
[Reference](https://learn.microsoft.com/en-us/azure/cosmos-db/emulator).

And it's after reading up on it, we can then look at how we can get the certificate
from the emulator in order to then add it as a trusted
certificate: [Docs](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-develop-emulator?tabs=docker-linux%2Ccsharp&pivots=api-nosql#install-the-emulator)

## Dockerizing the service

We need to install the NuGet to connect to Cosmos first: `Microsoft.Azure.Cosmos`.

And right after we can spin up a container of the emulator:

**Windows**

```
$parameters = @(
    "--publish", "8081:8081"
    "--publish", "10250-10255:10250-10255"
    "--name", "windows-emulator"
    "--detach"
    "--tmpfs", "/data"
    "--env", "AZURE_COSMOS_EMULATOR_IP_ADDRESS_OVERRIDE=127.0.0.1"
)
docker run @parameters mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator:latest
```

**Linux**

```
docker run \
    --publish 8081:8081 \
    --publish 10250-10255:10250-10255 \
    --name linux-emulator \
    --detach \
    --env AZURE_COSMOS_EMULATOR_IP_ADDRESS_OVERRIDE=127.0.0.1
    mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator:latest
```

This should just about enough to start working with the **Windows** emulator, this is 
the original one that runs on a Windows container (it's a bit slower also). Bue we can  
use the `Linux` emulator as an alternative:

```
docker run --detach --publish 8081:8081 --publish 1234:1234 mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator:vnext-preview --protocol https
```

Sources:

- [Linux Emulator](https://learn.microsoft.com/en-us/azure/cosmos-db/emulator-linux)
- [Windows Emulator](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-develop-emulator?tabs=docker-linux%2Ccsharp&pivots=api-nosql#install-the-emulator)
- [What's Cosmos DB Emulator?](https://learn.microsoft.com/en-us/azure/cosmos-db/emulator)

## Running the service

By default, all emulator instances use the same credentials:

| Field             | Value                                                                                                                                        |
|-------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| Endpoint          | localhost:8081                                                                                                                               |
| Key               | C2y6yDjf5/R+ob0N8A7Cgv30VRDJIWEHLM+4QDU5DE2nQ9nDuVTqobD4b8mGGyPMbIZnqyMsEcaGQy67XIw/Jw==                                                     |
| Connection string | AccountEndpoint=https://localhost:8081/;AccountKey=C2y6yDjf5/R+ob0N8A7Cgv30VRDJIWEHLM+4QDU5DE2nQ9nDuVTqobD4b8mGGyPMbIZnqyMsEcaGQy67XIw/Jw==; |

And so the endpoint `/cosmos` will make use of these settings to attempt to connect 
to an instance and run some commands to create dummy data and run an upsert.

We are focusing on a **Docker image** that should be able to connect normally to 
our local instance, and so the repository has an `emulator.cert` file already present, 
the image should take it and then save it as trusted.

We also have to install Newtownsoft since it's a hard dependency: [Reference](https://github.com/Azure/azure-cosmos-dotnet-v3/issues/4900) 

The command to build the image will be (by standing at the root of the solution folder):

```
docker build -t dockerized-cosmos-connect -f ./WebApi/Dockerfile .
```

**IMPORTANT:** Even though the docs mention a whole process for importing an auto-signed 
certificate and trusting it, this doesn't work, the client may it be local or 
in container will never accept the emulator's certificate. So only for _dev_ purposes, 
we should configure the connection to skip over the SSL validation alongside it 
being in Gateway mode:

```csharp
using CosmosClient client = new(
        accountEndpoint: config["Cosmos:AccountEndpoint"],
        authKeyOrResourceToken: config["Cosmos:AuthKey"],
        new CosmosClientOptions
        {
            SerializerOptions = new CosmosSerializationOptions
            {
                PropertyNamingPolicy = CosmosPropertyNamingPolicy.CamelCase
            },
            HttpClientFactory = () =>
            {
                /*                               *** WARNING ***
                 * This code is for demo purposes only. In production, you should use the default behavior,
                 * which relies on the operating system's certificate store to validate the certificates.
                 */
                HttpMessageHandler httpMessageHandler = new HttpClientHandler
                {
                    ServerCertificateCustomValidationCallback = HttpClientHandler.DangerousAcceptAnyServerCertificateValidator
                };
                return new HttpClient(httpMessageHandler);
            },
            ConnectionMode = ConnectionMode.Gateway
        }
    );
```
**This configuration is a must**

This will take care of no longer showing an SSL error, nor when trying to run any 
sort of command it just hanging up and returning some cryptic error about a resource 
no longer available. 

## Docker-compose

The Cosmos Emulator, definitely does things in the background depending on as to how 
you initialize it, since its certificate that's auto-generated has a different 
DNS alternate name based on the hostname [Reference](https://github.com/Azure/azure-cosmos-db-emulator-docker/issues/76). 

And so, the docker-compose setup so that we can get a fully orchestrated environment 
working will require for specific configurations:

```yaml
networks:
  default:
    external: false
    ipam:
      driver: default
      config:
        - subnet: "172.16.238.0/24"

services:
  cosmosdb:
    restart: always
    container_name: "azure-cosmosdb-emulator"
    hostname: "azurecosmosemulator"
    image: 'mcr.microsoft.com/cosmosdb/linux/azure-cosmos-emulator:latest'
    mem_limit: 4GB
    tty: true
    ports:
      - '8081:8081' # Data Explorer
      - '8900:8900'
      - '8901:8901'
      - '8902:8902'
      - '10250:10250'
      - '10251:10251'
      - '10252:10252'
      - '10253:10253'
      - '10254:10254'
      - '10255:10255'
      - '10256:10256'
      - '10350:10350'
    environment:
      - AZURE_COSMOS_EMULATOR_PARTITION_COUNT=3
      - AZURE_COSMOS_EMULATOR_ENABLE_DATA_PERSISTENCE=false
      - AZURE_COSMOS_EMULATOR_IP_ADDRESS_OVERRIDE=172.16.238.246
    networks:
      default:
        ipv4_address: 172.16.238.246
  webapi:
    image: webapi
    build:
      context: .
      dockerfile: WebApi/Dockerfile
    depends_on:
      - cosmosdb
    ports:
      - '5050:8080'
    environment:
      - Cosmos__AccountEndpoint=https://azurecosmosemulator:8081/
    networks:
      default:
        ipv4_address: 172.16.238.242
```
This makes use of the Windows image for cosmos, it also sets up its own virtual 
network so that we can connect successfully from our api container to the cosmos 
backing service. The important bits are the `hostname` plus its `AZURE_COSMOS_EMULATOR_IP_ADDRESS_OVERRIDE` 
value that should be the same as the default address of the container.

It's by configuring everything this way, that we can manage to get a Cosmos Emulator 
up and running in a container and shaping its behavior however we see fit (the 
docker-compose approach enables for us to customize even further, yet, as mentioned 
before it has its cons this being a windows container).

## References

- [Solution for hanging and SSL validation](https://github.com/Azure/azure-cosmos-db-emulator-docker/issues/135)
- [Mention of the linux emulator](https://github.com/Azure/azure-cosmos-db-emulator-docker/issues/76)
- [Great reference](https://github.com/Azure/cosmosdb-emulator-recipes/tree/main/dotnet/linux)
- [How to use Cosmos](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/how-to-dotnet-get-started)
- [Instructions on repo](https://github.com/Azure/azure-cosmos-db-emulator-docker/blob/master/README.md)
- [Troubleshooting the emulator](https://learn.microsoft.com/en-us/troubleshoot/azure/cosmos-db/tools-connectors/emulator)
- [About endpoint to retrieve certificate](https://github.com/Azure/azure-cosmos-db-emulator-docker/issues/117)
- [Consuming .exe Emulator (Windows)](https://stackoverflow.com/questions/55018321/azure-cosmos-db-emulator-on-a-local-area-network)
- [Mention of hanging up](https://stackoverflow.com/questions/71606631/azure-cosmos-emulator-hangs-when-i-try-to-create-a-database)
- [In regards to env variables](https://github.com/Azure/azure-cosmos-dotnet-v3/issues/2623)
- [Mention of the right way to use it](https://github.com/Azure/azure-cosmos-dotnet-v3/issues/1232#issuecomment-670673733)