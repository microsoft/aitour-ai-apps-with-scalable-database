# Demo: Retrieval Augmented Generation (RAG) with Azure Cosmos DB for NoSQL

Demo shows a multi-tenant, multi-user, Generative-AI RAG Pattern application using Azure Cosmos DB for NoSQL with its new vector database capabilities with Azure OpenAI Service on Azure App Service. 


Services Used:
- Azure Cosmos DB for NoSQL
- Azure OpenAI Service
- Azure App Service


## Setup:

### 1. Clone repo and configure for RAG

[Clone the repo](https://github.com/AzureCosmosDB/cosmosdb-nosql-copilot).

**To enable rag in the application**: Clone the repo, and open up the `src > services > ChatService.cs` file. 

Replace it with the following:

```
using Cosmos.Copilot.Models;
using Microsoft.ML.Tokenizers;

namespace Cosmos.Copilot.Services;

public class ChatService
{

    private readonly CosmosDbService _cosmosDbService;
    private readonly OpenAiService _openAiService;
    private readonly SemanticKernelService _semanticKernelService;
    private readonly int _maxConversationTokens;
    private readonly double _cacheSimilarityScore;
    private readonly int _productMaxResults;

    public ChatService(CosmosDbService cosmosDbService, OpenAiService openAiService, SemanticKernelService semanticKernelService, string maxConversationTokens, string cacheSimilarityScore, string productMaxResults)
    {
        _cosmosDbService = cosmosDbService;
        _openAiService = openAiService;
        _semanticKernelService = semanticKernelService;

        _maxConversationTokens = Int32.TryParse(maxConversationTokens, out _maxConversationTokens) ? _maxConversationTokens : 100;
        _cacheSimilarityScore = Double.TryParse(cacheSimilarityScore, out _cacheSimilarityScore) ? _cacheSimilarityScore : 0.99;
        _productMaxResults = Int32.TryParse(productMaxResults, out _productMaxResults) ? _productMaxResults: 10;
    }

    public async Task InitializeAsync()
    {
        await _cosmosDbService.LoadProductDataAsync();
    }

    /// <summary>
    /// Returns list of chat session ids and names for left-hand nav to bind to (display Name and ChatSessionId as hidden)
    /// </summary>
    public async Task<List<Session>> GetAllChatSessionsAsync(string tenantId, string userId)
    {
        return await _cosmosDbService.GetSessionsAsync(tenantId,userId);
    }

    /// <summary>
    /// Returns the chat messages to display on the main web page when the user selects a chat from the left-hand nav
    /// </summary>
    public async Task<List<Message>> GetChatSessionMessagesAsync(string tenantId, string userId,string? sessionId)
    {
        ArgumentNullException.ThrowIfNull(tenantId);
        ArgumentNullException.ThrowIfNull(userId);
        ArgumentNullException.ThrowIfNull(sessionId);

        return await _cosmosDbService.GetSessionMessagesAsync(tenantId,userId, sessionId); ;
    }

    /// <summary>
    /// User creates a new Chat Session.
    /// </summary>
    public async Task<Session> CreateNewChatSessionAsync(string tenantId, string userId)
    {

        Session session = new(tenantId, userId);

        await _cosmosDbService.InsertSessionAsync(tenantId,  userId,session);

        return session;

    }

    /// <summary>
    /// Rename the Chat Session from "New Chat" to the summary provided by OpenAI
    /// </summary>
    public async Task RenameChatSessionAsync(string tenantId, string userId,string? sessionId, string newChatSessionName)
    {
        ArgumentNullException.ThrowIfNull(tenantId);
        ArgumentNullException.ThrowIfNull(userId);
        ArgumentNullException.ThrowIfNull(sessionId);

        Session session = await _cosmosDbService.GetSessionAsync(tenantId,  userId,sessionId);

        session.Name = newChatSessionName;

        await _cosmosDbService.UpdateSessionAsync(tenantId, userId,session);
    }

    /// <summary>
    /// User deletes a chat session
    /// </summary>
    public async Task DeleteChatSessionAsync(string tenantId, string userId, string? sessionId)
    {
        ArgumentNullException.ThrowIfNull(tenantId);
        ArgumentNullException.ThrowIfNull(userId);
        ArgumentNullException.ThrowIfNull(sessionId);

        await _cosmosDbService.DeleteSessionAndMessagesAsync( tenantId, userId, sessionId);
    }

    /// <summary>
    /// Get a completion for a user prompt from Azure OpenAi Service
    /// This is the main LLM Workflow for the Chat Service
    /// </summary>
    public async Task<Message> GetChatCompletionAsync(string tenantId, string userId, string? sessionId, string promptText)
    {
        ArgumentNullException.ThrowIfNull(tenantId);
        ArgumentNullException.ThrowIfNull(userId);
        ArgumentNullException.ThrowIfNull(sessionId);

        //Create a message object for the new User Prompt and calculate the tokens for the prompt
        Message chatMessage = await CreateChatMessageAsync(tenantId,  userId, sessionId, promptText);

        //Grab context window from the conversation history up to the maximum configured tokens
        List<Message> contextWindow = await GetChatSessionContextWindow( tenantId,  userId, sessionId);

        //Perform a cache search to see if this prompt has already been used in the same context window as this conversation
        (string cachePrompts, float[] cacheVectors, string cacheResponse) = await GetCacheAsync(contextWindow);

        //Cache hit, return the cached completion
        if (!string.IsNullOrEmpty(cacheResponse))
        {
            chatMessage.Completion = cacheResponse;
            chatMessage.Completion += " (cached response)";
            chatMessage.CompletionTokens = 0;

            //Persist the prompt/completion, update the session tokens
            await UpdateSessionAndMessage( tenantId,  userId, sessionId, chatMessage);

            return chatMessage;
        }
        else  //Cache miss, send to OpenAI to generate a completion
        {
            //Generate embeddings for the user prompt
            float[] promptVectors = await _openAiService.GetEmbeddingsAsync(promptText);

            //These functions are just for simple completions without RAG Pattern
            //Generate a completion and tokens used with current context window (just user prompts, no vector search results, non RAG Pattern)
            //(chatMessage.Completion, chatMessage.CompletionTokens) = await _openAiService.GetChatCompletionAsync(sessionId, contextWindow);
            //(chatMessage.Completion, chatMessage.CompletionTokens) = await _semanticKernelService.GetChatCompletionAsync(sessionId, contextWindow);

            //These functions are for doing RAG Pattern completions
            //Uncomment SearchProductsAsync() and one of the GetRagCompletionAsync() functions to do RAG Pattern completions
            //Perform vector search for products
            List<Product> products = await _cosmosDbService.SearchProductsAsync(promptVectors, _productMaxResults);

            //Generate a completion and tokens used from current context window and vector search results
            (chatMessage.Completion, chatMessage.CompletionTokens) = await _openAiService.GetRagCompletionAsync(sessionId, contextWindow, products);
            //(chatMessage.Completion, chatMessage.CompletionTokens) = await _semanticKernelService.GetRagCompletionAsync(sessionId, contextWindow, products);

            //Cache the prompts in the current context window and their vectors with the generated completion
            await CachePutAsync(cachePrompts, cacheVectors, chatMessage.Completion);
        }


        //Persist the prompt/completion, update the session tokens
        await UpdateSessionAndMessage( tenantId, userId, sessionId, chatMessage);

        return chatMessage;
    }

    /// <summary>
    /// Get the context window for this conversation. This is used in cache search as well as generating completions
    /// </summary>
    private async Task<List<Message>> GetChatSessionContextWindow(string tenantId, string userId, string sessionId)
    {
        ArgumentNullException.ThrowIfNull(tenantId);
        ArgumentNullException.ThrowIfNull(userId);
        ArgumentNullException.ThrowIfNull(sessionId);

        int? tokensUsed = 0;

        List<Message> allMessages = await _cosmosDbService.GetSessionMessagesAsync(tenantId, userId, sessionId);
        List<Message> contextWindow = new List<Message>();

        //Start at the end of the list and work backwards
        //This includes the latest user prompt which is already cached
        for (int i = allMessages.Count - 1; i >= 0; i--)
        {
            tokensUsed += allMessages[i].PromptTokens + allMessages[i].CompletionTokens;

            if (tokensUsed > _maxConversationTokens)
                break;

            contextWindow.Add(allMessages[i]);
        }

        //Invert the chat messages to put back into chronological order 
        contextWindow = contextWindow.Reverse<Message>().ToList();

        return contextWindow;

    }

    /// <summary>
    /// Use OpenAI to summarize the conversation to give it a relevant name on the web page
    /// </summary>
    public async Task<string> SummarizeChatSessionNameAsync(string tenantId, string userId, string? sessionId)
    {
        ArgumentNullException.ThrowIfNull(tenantId);
        ArgumentNullException.ThrowIfNull(userId);
        ArgumentNullException.ThrowIfNull(sessionId);

        //Get the messages for the session
        List<Message> messages = await _cosmosDbService.GetSessionMessagesAsync( tenantId,  userId, sessionId);

        //Create a conversation string from the messages
        string conversationText = string.Join(" ", messages.Select(m => m.Prompt + " " + m.Completion));

        //Send to OpenAI to summarize the conversation
        string completionText = await _openAiService.SummarizeAsync(sessionId, conversationText);
        //string completionText = await _semanticKernelService.SummarizeConversationAsync(conversationText);

        await RenameChatSessionAsync( tenantId,  userId, sessionId, completionText);

        return completionText;
    }

    /// <summary>
    /// Add user prompt to a new chat session message object, calculate token count for prompt text.
    /// </summary>
    private async Task<Message> CreateChatMessageAsync(string tenantId, string userId, string sessionId, string promptText)
    {
        ArgumentNullException.ThrowIfNull(tenantId);
        ArgumentNullException.ThrowIfNull(userId);
        ArgumentNullException.ThrowIfNull(sessionId);

        //Calculate tokens for the user prompt message.
        int promptTokens = GetTokens(promptText);

        //Create a new message object.
        Message chatMessage = new(tenantId, userId, sessionId, promptTokens, promptText, "");

        await _cosmosDbService.InsertMessageAsync(tenantId, userId, chatMessage);

        return chatMessage;
    }

    /// <summary>
    /// Update session with user prompt and completion tokens and update the cache
    /// </summary>
    private async Task UpdateSessionAndMessage(string tenantId, string userId, string sessionId, Message chatMessage)
    {
        ArgumentNullException.ThrowIfNull(tenantId);
        ArgumentNullException.ThrowIfNull(userId);
        ArgumentNullException.ThrowIfNull(sessionId);

        //Update the tokens used in the session
        Session session = await _cosmosDbService.GetSessionAsync(tenantId, userId, sessionId);
        session.Tokens += chatMessage.PromptTokens + chatMessage.CompletionTokens;

        //Insert new message and Update session in a transaction
        await _cosmosDbService.UpsertSessionBatchAsync( tenantId,  userId, session, chatMessage);

    }

    /// <summary>
    /// Calculate the number of tokens from the user prompt
    /// </summary>
    private int GetTokens(string userPrompt)
    {

        Tokenizer _tokenizer = Tokenizer.CreateTiktokenForModel("gpt-3.5-turbo");

        return _tokenizer.CountTokens(userPrompt);

    }

    /// <summary>
    /// Query the semantic cache with user prompt vectors for the current context window in this conversation
    /// </summary>
    private async Task<(string cachePrompts, float[] cacheVectors, string cacheResponse)> GetCacheAsync(List<Message> contextWindow)
    {
        //Grab the user prompts for the context window
        string prompts = string.Join(Environment.NewLine, contextWindow.Select(m => m.Prompt));

        //Get the embeddings for the user prompts
        float[] vectors = await _openAiService.GetEmbeddingsAsync(prompts);
        //float[] vectors = await _semanticKernelService.GetEmbeddingsAsync(prompts);

        //Check the cache for similar vectors
        string response = await _cosmosDbService.GetCacheAsync(vectors, _cacheSimilarityScore);

        return (prompts, vectors, response);
    }

    /// <summary>
    /// Cache the last generated completion with user prompt vectors for the current context window in this conversation
    /// </summary>
    private async Task CachePutAsync(string cachePrompts, float[] cacheVectors, string generatedCompletion)
    {
        //Include the user prompts text to view. They are not used in the cache search.
        CacheItem cacheItem = new(cacheVectors, cachePrompts, generatedCompletion);

        //Put the prompts, vectors and completion into the cache
        await _cosmosDbService.CachePutAsync(cacheItem);
    }

    /// <summary>
    /// Clear the Semantic Cache
    /// </summary>
    public async Task ClearCacheAsync()
    {
        await _cosmosDbService.CacheClearAsync();
    }
}
```

### 1. Deploy
Follow the instructions to deploy the sample in the [demo repository](https://github.com/AzureCosmosDB/cosmosdb-nosql-copilot?tab=readme-ov-file#instructions). It instructs you to clone the repo and then use `azd up` to begin deployment. Either complete this step in the Azure Cloud Shell or use the [Azure CLI]() to complete on a local machine.  Be sure to make the change in the previous step to enable RAG.

You should now have the following resources deployed: 
- Resource group named `CosmosNoSQLRAGDemo`
- Azure App Service .NET Web App named `web-XXXX`.
- App Service plan named `plan-XXXX`
- Managed Identity named `ua-id-XXXX`
- Azure Cosmos DB for NoSQL Account named `cosmos-XXXX` with the feature `Vector Search for NoSQL API (preview)` set to `On`
    - Database named `cosmoscopilotdb` with containers `cache`, `chat`, and `products`
- Azure Open AI Service named `openai-XXXX` with
    - Deployed GPT 4o model named `gpt-4o`
    - Deployed `text-embedding-3-large` model named `text-3-large`

## Presenting the Demo

### Setup
It's recommended to have 2 tabs open in your browser for this demo:
    - Azure Portal OR cosmos.azure.com Cosmos DB data explorer with the `cosmoscopilotdb` database selected.
    - Deployed chat app

**(Optional)** Try some prompts to generate data for the database.

**Note**: If you have been testing previously, you may want to clear the cache (top right of the app, or delete all the documents in the cache container)

### 1. Describe the scenario (optionally, show the generated data in Data Explorer):

```
Let's look at a full end to end RAG scenario with Azure Cosmos DB for NoSQL. It's is a multitenant and multiuser app that is a knowledge base for biking products.
We're using vector capabilities in 2 distinct ways: For vector search using the RAG pattern and also as a semantic cache to store previous answers. Let's learn more about the cache. 

Generating completions from an LLM are computationally expensive in time and cost. These metrics increase as the amount of text increases, especially when applying the RAG pattern.We can create a cache for this type of solution to reduce both cost and latency.

So instead of immediately sending the prompt to the completions model, we vectorize the query and check the cache first. If there's a match the response will be very fast. If not, the query will be sent to the completions model.

We're applying RAG here by including a product catalog for a bike shop as part of the generation. Users can ask questions about bikes and biking accessories in the catalog. The user's prompt is vectorized and used to execute a vector search against the vectorized product data in the catalog. The most relevant items are returned then passed to the LLM to generate a response.

Let's explore how they work in the chat app.

```

### 2. Ask a Question in the Chat

`Let's check what's in our product catalog`

- Start with `What bikes do you have?`, wait for response
- Switch to data explorer

```
 Behind the scenes 3 things happened: 
 1. We stored the session history in the chat container (show chat container, you can ask Query Advisor/Copilot for Cosmos DB to show the most recent entries).

 2. We checked the cache to see if the same question has been asked. The vector query compares the vectors in the cache and returns the best result above the similarity score filter. The similarity score is set to .99, so the queries must be very similar. If there's no match, the query is sent to the completion model. (show cache container)

 3. We finally vectorize the prompt and store it along with the completion.
```

### 3. RAG Walkthrough
- Return to app

```
Let's see if there's products other than bikes in here
```

- Ask `What products do you have?`, wait for response

```
While it's great to see what's available. Applying the RAG pattern would be especially helpful here If I wanted to make a purchase here with a specified budget, the completions model could give me some recommendations. Let's try it.
```

- Ask `I'd like to get a bike and 2 pairs of bike shorts. I have a budget of 2000`
- Then ask `Is there a more expensive bike I could purchase within my $2000 budget?`
- Feel free to try other combinations.

### 3. Semantic Cache

```
We mentioned before that our queries are saved in the cache, let's take a look at how fast our response is to a previous question.
```

- Ask a previous question like `What bikes do you have?` (try to make it as similar as possible to a previous question, similarity score is .99!)
- Wait for response, the reply will end with the note `(cached response)`

### 4. Wrap up

```
With Azure Cosmos DB, we're able to build a efficient intelligent app that revolutionizes the way we interact with data. With clever techniques like RAG and semantic caching we can streamline interactions with data and embed a layer of smart, responsive features. 
```