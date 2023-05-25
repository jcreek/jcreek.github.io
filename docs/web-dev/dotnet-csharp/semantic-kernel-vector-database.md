---
tags:
  - web development
  - dotnet
  - c#
  - AI
  - GPT
  - SK
  - semantic kernel
---

# Semantic Kernel (GPT with C#) - using a vector database to store embeddings

_2023-05-25_

At the time of writing there isn't any documentation on using vector databases to store embeddings, so I'm sharing my notes from my time digging around in the semantic kernel code.

## A brief explanation

So, what am I even talking about? As the global phenomenon that is Chat GPT has shown, the OpenAI GPT models are capable of making some impressively powerful tools. However, even as an experienced developer, working with LLM AIs is not something I nor, most of my colleagues have experience of. This is where the semantic kernel comes in. It's essentially a wrapper that allows you to use the GPT models in a more familiar way, using C# (or Python). It's a great tool for developers who want to use the power of GPT, but don't want to have to learn how to use it directly. 

There's some excellent resources available covering how you can create a text file that serves as a 'skill' - a set of prompts and responses that the AI can use to generate text - and for using volatile memory to store 'embeddings'. I won't go into those here, but the official Microsoft documentation is a great place to start.

Embeddings are essentially a way of storing a representation of a piece of text. The semantic kernel uses them to store the prompts and responses that you provide, and then uses them to find the most similar text to a given input. This is how it can generate responses to a given input. When you start trying to add context to a skill, you'll quickly find that the amount of prompts and responses you need to add to cover all the possible inputs is huge. This is where a vector database comes in.

A vector database is different to a relational or document database. I won't explain the differences here, but they can be used to store data in a way that best enables you to programmatically find the most similar data to a given input. In this case, I'm going to store contextual data that I want my skill to use to generate responses. I'll then use the vector database to provide this data to my skill, in a way that will not exceed the token limits of the GPT model.

## Setting up a console app to make use of semantic kernel

There's already excellent documentation on how to do this, so I won't go into it here. I'll assume you've already got a console app set up and ready to go, and that you've added the semantic kernel NuGet package using `dotnet add package Microsoft.SemanticKernel --version 0.14.547.1-preview` - take a look at the [official documentation](https://docs.microsoft.com/en-us/semantic-kernel/get-started) if you haven't.

## Setting up the vector database to store embeddings

This is remarkably easy thanks to some NuGet packages that have been provided alongside the semantic kernel. I'm going to use the SQLite database, but there are other, more robust options available.

You'll need to install the relevant NuGet package, which in our case is the SQLite one - `dotnet add package Microsoft.SemanticKernel.Connectors.Memory.Sqlite --version 0.14.547.1-preview` will do that for you.

First, set up a new kernel, enabling embeddings, and connecting to your SQLite database. You'll need to provide a connection string, which is just a path to the database file. I've used a relative path here, but you can use an absolute path if you prefer. I've left in a commented out line to demonstrate what setting this up with a standard in-memory database would look like. 

```csharp linenums="1"
var kernel = new KernelBuilder()
    .Configure(c =>
    {
        if (useAzureOpenAI)
        {
            c.AddAzureTextEmbeddingGenerationService("text-embedding-ada-002", azureEndpoint, apiKey);
            c.AddAzureTextCompletionService(model, azureEndpoint, apiKey);
        }
        else
        {
            c.AddOpenAITextEmbeddingGenerationService("text-embedding-ada-002", apiKey);
            c.AddOpenAITextCompletionService(model, apiKey, orgId);
        }
    })
    .WithMemoryStorage(await SqliteMemoryStore.ConnectAsync("memory.sqlite"))
    // .WithMemoryStorage(new VolatileMemoryStore())
    .Build();
```

Next I'm going to set up a dictionary of GitHub URLs and their descriptions. I'm going to use these to demonstrate how to store embeddings in the vector database. I've used a dictionary here, but you could use any collection of key-value pairs.

```csharp linenums="1"
const string memoryCollectionName = "SKGitHub";

var githubFiles = new Dictionary<string, string>()
{
    ["https://github.com/microsoft/semantic-kernel/blob/main/README.md"]
        = "README: Installation, getting started, and how to contribute",
    ["https://github.com/microsoft/semantic-kernel/blob/main/samples/notebooks/dotnet/02-running-prompts-from-file.ipynb"]
        = "Jupyter notebook describing how to pass prompts from a file to a semantic skill or function",
    ["https://github.com/microsoft/semantic-kernel/blob/main/samples/notebooks/dotnet/00-getting-started.ipynb"]
        = "Jupyter notebook describing how to get started with the Semantic Kernel",
    ["https://github.com/microsoft/semantic-kernel/tree/main/samples/skills/ChatSkill/ChatGPT"]
        = "Sample demonstrating how to create a chat skill interfacing with ChatGPT",
    ["https://github.com/microsoft/semantic-kernel/blob/main/dotnet/src/SemanticKernel/Memory/Volatile/VolatileMemoryStore.cs"]
        = "C# class that defines a volatile embedding store",
    ["https://github.com/microsoft/semantic-kernel/tree/main/samples/dotnet/KernelHttpServer/README.md"]
        = "README: How to set up a Semantic Kernel Service API using Azure Function Runtime v4",
    ["https://github.com/microsoft/semantic-kernel/tree/main/samples/apps/chat-summary-webapp-react/README.md"]
        = "README: README associated with a sample starter react-based chat summary webapp",
};
```

Obviously these have to be stored in the vector database before they can be used. I'm going to use the `SaveReferenceAsync` method to do this. This method takes a collection name, a description, the text to be stored, and an external ID and source name. The external ID is a unique identifier for the piece of text, and the source name is a string that identifies the source of the text. I'm using the GitHub URL as the external ID, and the source name as 'GitHub'. You can use whatever you like here, but it's important that the external ID is unique. I'm also going to print out a message for each URL that's saved, so I can see what's happening.

```csharp linenums="1"
Console.WriteLine("Adding some GitHub file URLs and their descriptions to a volatile Semantic Memory.");
var i = 0;
foreach (var entry in githubFiles)
{
    await kernel.Memory.SaveReferenceAsync(
        collection: memoryCollectionName,
        description: entry.Value,
        text: entry.Value,
        externalId: entry.Key,
        externalSourceName: "GitHub"
    );
    Console.WriteLine($"  URL {++i} saved");
}
```

## Querying the vector database

Now that we've stored some data in the vector database, we can query it. I'm going to use the `SearchAsync` method to do this. This method takes a collection name, a query string, and a limit and minimum relevance score. The limit is the maximum number of results to return, and the minimum relevance score is the minimum relevance score that a result must have to be returned. I'm going to set the limit to 5, and the minimum relevance score to 0.77. I'm also going to print out the query string, so I can see what's happening.

The reason I want to limit the number of results is that the vector database will return all results that have a relevance score above the minimum relevance score. If I didn't limit the number of results, I'd get all the results that have a relevance score above 0.77, which would be all of them. It is critical to ensure that we stay below the token limit for the OpenAI API for the 'ask' that we're going to perform, and finding only a limited set of the most relevant data to use for the 'ask' is a good way to do this.

```csharp linenums="1"
string ask = "I love Jupyter notebooks, how should I get started?";
Console.WriteLine("===========================\n" +
                    "Query: " + ask + "\n");

var memories = kernel.Memory.SearchAsync(memoryCollectionName, ask, limit: 5, minRelevanceScore: 0.77);

i = 0;
await foreach (MemoryQueryResult memory in memories)
{
    Console.WriteLine($"Result {++i}:");
    Console.WriteLine("  URL:     : " + memory.Metadata.Id);
    Console.WriteLine("  Title    : " + memory.Metadata.Description);
    Console.WriteLine("  Relevance: " + memory.Relevance);
    Console.WriteLine();
}
```

Running this ask gives the following results:

```
===========================
Query: I love Jupyter notebooks, how should I get started?

Result 1:
  URL:     : https://github.com/microsoft/semantic-kernel/blob/main/samples/notebooks/dotnet/00-getting-started.ipynb
  Title    : Jupyter notebook describing how to get started with the Semantic Kernel
  Relevance: 0.8678345730903347

Result 2:
  URL:     : https://github.com/microsoft/semantic-kernel/blob/main/samples/notebooks/dotnet/02-running-prompts-from-file.ipynb
  Title    : Jupyter notebook describing how to pass prompts from a file to a semantic skill or function
  Relevance: 0.8163350291107238

Result 3:
  URL:     : https://github.com/microsoft/semantic-kernel/blob/main/README.md
  Title    : README: Installation, getting started, and how to contribute
  Relevance: 0.8084409058091631
```

It has found the three GitHub URLs that are most relevant to the query string, and returned them in order of relevance. The first result is the Jupyter notebook that describes how to get started with the Semantic Kernel, which is exactly what I was looking for.

## Conclusion

This is only an incredibly basic example, but hopefully it's enough to get you kick-started with the Semantic Kernel and using a vector database to store embeddings. 
