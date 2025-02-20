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
from the emulator in order to then add it as a trusted certificate: [Docs](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-develop-emulator?tabs=docker-linux%2Ccsharp&pivots=api-nosql#install-the-emulator)

## Dockerizing the service

We need to install the NuGet to connect to Cosmos first: `Microsoft.Azure.Cosmos`.

