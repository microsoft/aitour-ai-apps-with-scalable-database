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

### 4. Transition to from Query Editor to Azure AI Search

```
Right here in the portal, there’s a button to set up Azure AI Search. With just a few clicks, we can connect our SQL data to Azure AI Search, enriching our database with AI capabilities.

Azure SQL Database has vector search capabilities all on its own, but with Azure AI search we can apply cognitive skills to our data or extract key information to make our data work smarter.

We already have an index set up so let's take a look in Azure AI Search.
```

### 4. Move on to Azure AI Search - Indexes Blade

#### (Optional) - Search Explorer

```
Here's our data, let's use the search explorer to find highly rated products, we define as a score greater than 4.
```

**Go to View > JSON View and paste in the query**

```
{
  "search": "{   \"search\": \"top rated products\",   \"searchFields\": \"parent_text, parent_summary\",   \"select\": \"id, parent_product_id, parent_text, parent_summary, parent_score\",   \"filter\": \"parent_score gt 4\",   \"orderby\": \"parent_score desc\" }",
  "answers": "extractive|count-3",
  "captions": "extractive",
  "queryLanguage": "en-US",
  "semanticConfiguration": "my-semantic-config",
  "queryType": "semantic",
  "vectorQueries": [
    {
      "kind": "text",
      "text": "{   \"search\": \"top rated products\",   \"searchFields\": \"parent_text, parent_summary\",   \"select\": \"id, parent_product_id, parent_text, parent_summary, parent_score\",   \"filter\": \"parent_score gt 4\",   \"orderby\": \"parent_score desc\" }",
      "fields": "vector"
    }
  ]
}
```


#### Fields View

```
The Fields view has all the columns we pulled in from our table. This data is configured so that we each review has unique identifier that can be easily searched and filtered with any tokenization. 

The parent text, summary, and score fields provide additional context and metadata about the scores.

Finally the The vector field stores vector representations of the review text to find semantically similar reviews. We'll look at how these were turned into vectors with skills but first let's talk about how we convert queries into vectors.
```

### Vector Profiles 

```
Vectorizers in a vector profile turn texts or images into vector representation during query execution. We use a deployed Azure OpenAI model to find semantically similar reviews.

Let's look at how the data that's stored is turned into vectors with skills.
``` 


### 5. Azure AI Search - Indexes Blade > sql-customer-index-skillset

```
Skillsets are used during the indexing process to transform the data into vector representations. We're using a vectorization skill to generate embeddings of the review content, and is stored in the vector field. Any new data that is added to the SQL table would also go through the same vectorization process.

There's also another skill in here that splits the document text into chunks based on pages,  making it easier to generate embeddings, perform detailed searches, and the token limits are much easier to manage.

The data we saw in the table has been transformed into vectors and is now a knowledge base of product reviews. We can take it even further in Azure OpenAI Chat Playground and create an interactive experience where users can ask questions about the products and receive detailed, context-aware responses.
```


### 6. Azure OpenAI Studio Chat Playground

#### SETUP
**NOTE**: The system message will reset when you refresh the page. Consider adding the system message right before you deliver the session.

Add the system message:

```
The user is searching for a product matching their query.  Tell the user that after searching through our product database, you recommend the product described in the provided product review. Your answer should summarize the review text, include the product ID, and mention the score given in the review. Present the answer in a readable format that clearly lists the products.
```


```
In the playground I have set up the index as my source (show Add your data configuration) and a system message. Let's try asking a question.
```

Try these prompts: 
- `What are customers saying about the dog products?`
- `What food products do customers like as snacks?`
- `What are customers general sentiment about the oatmeal?`

The citations point to specific reviews from the response.

```
The data we saw in the table has been transformed into vectors and now is a knowledge base of product reviews in Azure OpenAI Studio. We could take it even further here and transform it into an app (show the Deploy button).

We can achieve all of this programmatically through APIs or directly in the Azure Portal, giving you the flexibility to choose the approach that best fits your needs.

```

### 7. Wrap up

```
We started with Copilot, using natural language to query our database to help us learn and optimize our database interactions.

Next, we enhanced our search capabilities with Azure AI Search index. 

Finally, we integrated our data with the Azure OpenAI Chat Playground. This allowed us to ask questions about our product reviews and receive detailed, context-aware responses. 

Solid integration across the Microsoft Intelligent Data platform makes your AI transformation much smoother and efficient. 
```