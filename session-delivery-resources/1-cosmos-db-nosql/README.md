# Demo: Retrieval Augmented Generation (RAG) with Azure Cosmos DB for NoSQL

Demo shows a multi-tenant, multi-user, Generative-AI RAG Pattern application using Azure Cosmos DB for NoSQL with its new vector database capabilities with Azure OpenAI Service on Azure App Service. 


Services Used:
- Azure Cosmos DB for NoSQL
- Azure OpenAI Service
- Azure App Service


## Setup:

### 1. Deploy
Follow the instructions to deploy the sample in the [demo repository](https://github.com/AzureCosmosDB/cosmosdb-nosql-copilot?tab=readme-ov-file#instructions). It instructs you to clone the repo and then use `azd up` to begin deployment. Either complete this step in the Azure Cloud Shell or use the [Azure CLI]() to complete on a local machine.  

You should now have the following resources deployed: 
    - Resource group named `CosmosNoSQLRAGDemo`
    - Azure App Service .NET Web App named `web-XXXX`.
    - App Service plan named `plan-XXXX`
    - Managed Identity named `ua-id-XXXX`
    - Azure Cosmos DB for NoSQL Account named `cosmos-XXXX` with the feature `Vector Search for NoSQL API (preview)` set to `On`
        Database named `cosmoscopilotdb` with containers `cache`, `chat`, and `products`
    - Azure Open AI Service named `openai-XXXX` with
        - Deployed GPT 4o Turbo model named `gpt-4o`
        - Deployed `text-embedding-3-large` model named `text-3-large`

### Presenting the Demo
