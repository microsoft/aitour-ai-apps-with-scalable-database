## Setup

### 1. Deploy

Click on the the Deploy to Azure button below to deploy services needed. Alternatively you can run the Bicep file.

You should now have the following resources deployed: 
    - Resource group names `rg-aitour-databases`
    - Azure SQL server named `aitour-dbserver-XXXXX` and a database named `reviews_demo`. 
    - Azure AI Search resource named `aitour-search-XXXXX`
    - Azure Open AI Service named `aitour-aoai-XXXXX` with
        - Deployed GPT 3.5 Turbo model named `completions`
        - Deployed `text-embedding-ada-002` model named `embeddings`

### 2. Add data and configure services

- 
- pip install -r requirements.txt
- Add service keys to .env file

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
