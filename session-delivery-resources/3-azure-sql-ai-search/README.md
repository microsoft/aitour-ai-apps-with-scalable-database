# DEMO 3: Retrieval Augmented Generation with Azure SQL Database and Azure AI Search

The demo showcases the seamless integration of Azure SQL Database as a Platform-as-a-Service (PaaS) offering within the Microsoft Intelligent Data Platform. It demonstrates the utilization of Azure SQL Database as a data source in Azure AI Studio, along with the application of a RAG pattern using Azure OpenAI Service to analyze customer product reviews.

Services Used:
- Azure SQL Server and Database
- Azure AI Search
- Azure OpenAI Service

## Setup:

### 1. Deploy

Click on the the Deploy to Azure button below to deploy services needed. Alternatively you can run the Bicep file.

You should now have the following services: 
    - Resource group names `rg-aitour-databases`
    - Azure SQL server named `aitour-dbserver-XXXXX` and a database named `reviews_demo`. 
    - Azure AI Search resource named `aitour-search-XXXXX`
    - Azure Open AI Service named `aitour-aoai-XXXXX` with
        - Deployed GPT 3.5 Turbo model named `completions`
        - Deployed text-embedding-ada-002 named `embedddings`

### 2. Add data and configure services

Run the python script to:
- Populate the `reviews_demo` database with a table named `customer_reviews` with 99 records.
- Enable change tracking on the database
- Set `customer_reviews` as AI Search datasource with fields configured 
- Create an index named `sql-customer-index` with 2 skillsets: chunking data and vectorization
- Indexer for the data 

Alternatively, you can do this in the notebook and test it out.

### 3. Chat with data in Azure OpenAI Studio Chat Playground

1. In Azure OpenAI Studio, open Chat and make sure your `completions` model is selected
1. Select Add your data and set fields to:
    - Select your AI Search as data source
    - Select deployed resource and index named `sql-customer-index`
    - Check `Add vector search...`
1. Select next and in Data management and set fields:
    - Search type `Vector`
1. Select next and select `API`
1. Select Review and Finish > Save and Close

### Presenting the Demo

NOTE: The system message will reset when you refresh the page. Consider adding the system message before starting the session.

Add the system message:

`The user is searching for a product matching their query.  Tell the user that after searching through our product database, you recommend the product described in the provided product review. Your answer should summarize the review text, include the product ID, and mention the score given in the review. Present the answer in a readable format that clearly lists the products.`


Try these prompts: 
- `What are customers saying about the dog products?`
- `What food products do customers like as snacks?`
- `What are customers general sentiment about the oatmeal?`

The citations point to specific reviews from the response.


