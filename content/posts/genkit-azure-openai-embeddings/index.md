+++
date = '2025-11-04T15:20:51+01:00'
title = 'Creating and Using Embeddings with Genkit Go and Azure OpenAI'
tags = ['go', 'genkit', 'azure-openai']
series = ['Genkit Go and Azure OpenAI']
series_order = 3
summary = """This post demonstrates how to create embeddings with Genkit Go using the Azure OpenAI plugin, and implements a practical vector search example with PostgreSQL and pgvector."""
+++

In the first two articles of this series, I developed an Azure OpenAI plugin for Genkit Go, Google's open-source AI development framework. Those samples focused on text generation using chat completion models. In this post, I'll demonstrate how the same plugin works seamlessly with embedding models for vector search scenarios.

{{< alert "circle-info" >}}
If you're not familiar with embeddings, these are numerical vector representations of text that capture semantic meaning. These vectors enable similarity calculations between texts, forming the foundation for [vector search](https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/vector-search) and retrieval-augmented generation (RAG) patterns. See [Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-foundry/openai/concepts/understand-embeddings) or [OpenAI's documentation](https://platform.openai.com/docs/guides/embeddings) for detailed explanations. 
{{< /alert >}}

## Creating Embeddings with Genkit Go

Genkit Go uses *embedders* for embedding generation, paralleling how text generation uses models. While `genkit.Generate()` handles text generation, `genkit.Embed()` handles embedding generation, accepting either an embedder instance or an embedding model name.

Recall that our Azure OpenAI plugin embeds Genkit's built-in OpenAI plugin:

```go
type AzureOpenAI struct {
    *oai.OpenAI
    APIKey          string
    TokenCredential azcore.TokenCredential
    BaseURL         string
    Deployment      string
}
```

Because all methods of the embedded `OpenAI` plugin are promoted to our plugin (except where we override them, like `Init()`), the `AzureOpenAI` plugin works identically to the `OpenAI` plugin for embedding generation: 

```go {hl_lines=["10-18"]}
embedderName := "text-embedding-3-small"
ctx := context.Background()

aoai := &AzureOpenAI{
    BaseURL: baseURL,
    APIKey:  apiKey,
}
g := genkit.Init(ctx, genkit.WithPlugins(aoai))

embedder := aoai.Embedder(g, embedderName)
if embedder == nil {
    log.Fatalf("failed to create embedder %s", embedderName)
}

res, err := genkit.Embed(ctx, g, ai.WithEmbedder(embedder), ai.WithTextDocs("Hello World!"))
if err != nil {
    log.Fatalf("failed to embed: %v", err)
}

fmt.Println(len(res.Embeddings[0].Embedding)) // should be 1536 for text-embedding-3-small
```

We obtain an embedder through the plugin's `Embedder()` method and pass it to `genkit.Embed()`. The resulting `[]float32` slice—the vector representing `Hello World!`—has a length of 1536, matching the dimensionality of Azure OpenAI's `text-embedding-3-small` model.

{{< alert >}}
Different embedding models produce vectors with different dimensions. For example, `text-embedding-3-large` produces 3072-dimensional vectors, while `text-embedding-ada-002` (the previous generation) produces 1536-dimensional vectors. Always ensure your vector database schema matches your chosen model's dimensions.
{{< /alert >}}

## Vector Search with Azure OpenAI and pgvector

{{< alert "circle-info" >}}
The following code uses Go 1.25.3 and Genkit Go 1.1.0. Find the complete sample on [GitHub](https://github.com/joergjo/genkit-go-samples/tree/main/aoai-pgvector).
{{< /alert >}}

To demonstrate a practical use case, I'll adapt the [`pgvector` Genkit Go sample](https://github.com/firebase/genkit/tree/main/go/samples/pgvector). This sample implements basic vector search using PostgreSQL with the `pgvector` extension, which adds vector operations and a native vector type to PostgreSQL. The original uses Google's models, but only minimal changes are needed for Azure OpenAI.

Genkit uses *retrievers* to abstract document retrieval logic for specific stores. In this sample, the retriever reads documents from a PostgreSQL table and finds the two most semantically similar documents based on vector similarity. The [`defineRetriever()` function](https://github.com/firebase/genkit/blob/11794e5de2fd73e302dfd92a478bf3717abb4fda/go/samples/pgvector/main.go#L126) encapsulates this logic:

```go
func defineRetriever(g *genkit.Genkit, db *sql.DB, embedder ai.Embedder, retOpts *ai.RetrieverOptions) ai.Retriever {
    f := func(ctx context.Context, req *ai.RetrieverRequest) (*ai.RetrieverResponse, error) {
        ...    
    }
    return genkit.DefineRetriever(g, api.NewName(provider, "shows"), retOpts, f)
}
```

This function remains unchanged—it accepts an embedder as a parameter, so we simply substitute the Azure OpenAI embedder for the Google AI embedder when calling `defineRetriever()`.

### Database Setup

I'm using Docker to run PostgreSQL with `pgvector`, with the following Compose file automating database initialization:

```yaml
services:
  db:
    image: pgvector/pgvector:pg17
    container_name: pgvector-db
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydb
    ports:
      - "5432:5432"
    volumes:
      - pg-data:/var/lib/postgresql/data
      - ./setup.sql:/docker-entrypoint-initdb.d/1-migrate.sql

volumes:
  pg-data:
```

The `setup.sql` script (mounted as `/docker-entrypoint-initdb.d/1-migrate.sql`) is nearly identical to the original sample, with one critical change: the vector dimension must match `text-embedding-3-small`'s 1536 dimensions:

```sql {hl_lines=8}
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE embeddings (
    show_id TEXT NOT NULL,
    season_number INTEGER NOT NULL,
    episode_id INTEGER NOT NULL,
    chunk TEXT,
    embedding vector(1536),
    PRIMARY KEY (show_id, season_number, episode_id)
);
```

### Switching to Azure OpenAI

Now we can adapt the sample for Azure OpenAI. First, update the embedding model name:

```go {hl_lines="3"}
const (
	provider     = "pgvector"
	embedderName = "text-embedding-3-small"
)
```

Next, initialize Genkit with our `AzureOpenAI` plugin:

```go
func main() {
    baseURL := os.Getenv("AZ_OPENAI_BASE_URL")
    apiKey := os.Getenv("AZ_OPENAI_API_KEY")
    if baseURL == "" || apiKey == "" {
        log.Fatal("export AZ_OPENAI_BASE_URL and AZ_OPENAI_API_KEY to run this sample")
    }

    flag.Parse()
    ctx := context.Background()
    aoai := &AzureOpenAI{
		BaseURL: baseURL,
		APIKey:  apiKey,
	}
	g := genkit.Init(ctx, genkit.WithPlugins(aoai))
    ...
}
```

I'm using Azure OpenAI `v1` here (no deployment name specified). To use `2024-10-21` instead, set the `Deployment` field to your deployment name.

The `run()` method contains the application's core logic: it parses command-line flags, generates and stores embeddings for documents in the database (if requested), and defines a Genkit flow for vector search. The original creates a Google AI embedder directly in `run()` using a package-level function. Since our plugin only exposes an embedder method on the `AzureOpenAI` struct, we add a plugin parameter to `run()`:

```go
func run(g *genkit.Genkit, aoai *AzureOpenAI) error {
    ...
}
```

Finally, create the embedder using our plugin:

```go {hl_lines="2"}
ctx := context.Background()
embedder := aoai.Embedder(g, embedderName)
if embedder == nil {
    return fmt.Errorf("failed to create embedder %s", embedderName)
}
```

### Running the Sample

Running this sample requires the Genkit CLI to trigger flows in the running application since it does not provide any UX. This approach is great for development, allowing you to test individual flows without building a full API or UI.

Start the database and application:

```bash
export AZ_OPENAI_BASE_URL='https://<your-resource>.openai.azure.com/openai/v1/'
export AZ_OPENAI_API_KEY='<your-azure-openai-api-key>'
export GENKIT_ENV='dev'

docker compose up -d
# The init flag triggers the embedding generation
go run . -init
```

From another terminal, trigger the vector search flow:

```bash
genkit flow:run askQuestion '{"Show": "La Vie", "Question": "Who gets divorced?"}'
genkit flow:run askQuestion '{"Show": "Best Friends", "Question": "Who does Alice love?"}' 
```

The vector search results aren't returned as flow output—the sample logs the raw Go structs to the console where your application runs, showing the retrieved documents and their similarity scores.

![Console output when running the flow](output.jpg)

## Summary

This post demonstrated how the Azure OpenAI plugin for Genkit Go supports embedding generation in addition to text generation. By leveraging Go's struct embedding, the `AzureOpenAI` plugin inherits the OpenAI plugin's embedder functionality without additional code. I adapted Genkit's `pgvector` sample to use Azure OpenAI's `text-embedding-3-small` model, showing how the plugin integrates with Genkit's retriever pattern for vector search. The key changes involved swapping the embedding model and ensuring the database schema matched the model's 1536-dimensional output. This demonstrates how the Azure OpenAI plugin works across Genkit's AI capabilities—from chat completion to embeddings—with a consistent API.