## Setup

### 1. Create Azure Resources

Create the following resources: 
    - Resource group:
    - Azure SQL server and database, basic tier and be sure to `allow all azure services` to access the database in network settings.
    - Azure AI Search resource
    - Azure Open AI Service with
        - Deployed `GPT 3.5 Turbo` model named `completions`
        - Deployed `text-embedding-ada-002` model named `embeddings`

### 2. Add data and configure services

- 
- pip install -r requirements.txt
- Add resource keys to .env file

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


**NOTE**: The system message will reset when you refresh the page. Consider adding the system message right before you deliver the session.

Add the system message:

```
The user is searching for a product matching their query.  Tell the user that after searching through our product database, you recommend the product described in the provided product review. Your answer should summarize the review text, include the product ID, and mention the score given in the review. Present the answer in a readable format that clearly lists the products.
```