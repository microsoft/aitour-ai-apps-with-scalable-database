# DEMO 3: Retrieval Augmented Generation with Azure SQL Database and Azure AI Search

The demo showcases the seamless integration of Azure SQL Database as a Platform-as-a-Service (PaaS) offering within the Microsoft Intelligent Data Platform. It demonstrates the utilization of Azure SQL Database as a data source in Azure AI Studio, along with the application of a RAG pattern using Azure OpenAI Service to analyze customer product reviews.

Services Used:
- Azure SQL Server and Database
- Azure AI Search
- Azure OpenAI Service

## Setup

### 1. Deploy

Click on the the Deploy to Azure button below to deploy services needed. Alternatively you can run the Bicep file.

You should now have the following resources deployed: 
    - Resource group names `rg-aitour-databases`
    - Azure SQL server named `aitour-dbserver-XXXXX` and a database named `reviews_demo`. 
    - Azure AI Search resource named `aitour-search-XXXXX`
    - Azure Open AI Service named `aitour-aoai-XXXXX` with
        - Deployed GPT 3.5 Turbo model named `completions`
        - Deployed `text-embedding-ada-002` model named `embedddings`

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

## Presenting the Demo

### Prepare

It's recommended to have 3 tabs open in your browser for this demo:
    - Azure Portal with Azure SQL Database Query Editor for the `customer_reviews` table selected
    - Azure Portal with Azure AI Search resource open to Indexes > `sql-customer-index`
    - Azure OpenAI Studio Chat Playground (make sure your Index is selected and system message added)


### 1. Describe the scenario:

```
Let's walk through how Azure SQL seamlessly integrates with Azure AI Services to create a Copilot for product suggestions based on customer reviews. 

In the database we have a collection of a variety of product reviews, let's explore the data with Copilot. We're curious to about any reviews for pet products, like a dog.
```

### 2. Use Copilot in Query editor to search for `Find all the product reviews that mention dogs`

It should return a this query: 

`
SELECT *
FROM dbo.customer_reviews
WHERE Text LIKE '%dogs%';
`

### 3. Run the query - it returns 3 rows. Explain what's happening:

```
We used natural language to describe what we're looking for, and Copilot describes why it generated this SQL query. This is great for situations when you're unsure how to put together a complex query or when you need to optimize your query. Copilot is designed to understand the context of your database to generate accurate queries by using your database metadata so things like tables, column names, primary keys, and foreign keys are taken into consideration when generating a query.

This transforms the way we interact with our data, making it more accessible and intuitive.
```

### 4. Transition to from Data Explorer to Azure AI Search

```
Right here in the portal, there’s a button to set up Azure AI Search. With just a few clicks, we can connect our SQL data to Azure AI Search, enriching our database with AI capabilities.

Azure SQL Database has vector search capabilities all on its own, but with Azure AI search we can apply cognitive skills to our data or extract key information to make our data work smarter.

We already have an index set up so let's take a look in Azure AI Search.
```

### 4. Move on to Azure AI Search - Indexes Blade





**NOTE**: The system message will reset when you refresh the page. Consider adding the system message right before you deliver the session.

Add the system message:

```
The user is searching for a product matching their query.  Tell the user that after searching through our product database, you recommend the product described in the provided product review. Your answer should summarize the review text, include the product ID, and mention the score given in the review. Present the answer in a readable format that clearly lists the products.
```


Try these prompts: 
- `What are customers saying about the dog products?`
- `What food products do customers like as snacks?`
- `What are customers general sentiment about the oatmeal?`

The citations point to specific reviews from the response.


