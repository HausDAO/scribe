# Knowledge Integration in Eliza Framework

This guide provides a comprehensive overview of how to integrate, manage, and utilize knowledge in the Eliza framework to create intelligent and informed agents.

## Table of Contents

- [Core Concepts](#core-concepts)
- [Knowledge Architecture](#knowledge-architecture)
- [Knowledge Types](#knowledge-types)
- [Setting Up Knowledge](#setting-up-knowledge)
- [Retrieval-Augmented Generation](#retrieval-augmented-generation)
- [Knowledge Integration API](#knowledge-integration-api)
- [Provider-Knowledge Integration](#provider-knowledge-integration)
- [Randomization for Variety](#randomization-for-variety)
- [Configuration Options](#configuration-options)
- [Best Practices](#best-practices)
- [Advanced Techniques](#advanced-techniques)
- [Troubleshooting](#troubleshooting)
- [Resources](#resources)

## Core Concepts

Knowledge integration in Eliza refers to how agents access, process, and utilize information beyond their inherent model capabilities:

- **Knowledge Base**: Collection of information accessible to agents
- **RAG**: Retrieval-Augmented Generation for dynamic information access
- **Semantic Search**: Finding relevant information based on meaning
- **Embedding**: Vector representations enabling semantic similarity matching

## Knowledge Architecture

### Core Components

The knowledge system comprises several key components:

1. **Basic Knowledge Module** (`knowledge.ts`): Simple document storage/retrieval
2. **RAGKnowledgeManager** (`ragknowledge.ts`): Advanced RAG implementation
3. **Embedding System**: Vector representations for semantic search
4. **Context Integration**: Combining retrieved knowledge with conversation context

### Implementation Flow

```
Documents → Preprocessing → Chunking → Embedding → Database Storage
                                                      ↓
Agent Response ← Context Assembly ← Knowledge Retrieval ← Query Processing
```

### Knowledge Structure

Each knowledge item includes:

```typescript
interface RAGKnowledgeItem {
    id?: UUID;                  // Unique identifier
    agentId?: UUID;             // Associated agent
    content: Content;           // Text and metadata
    embedding?: number[];       // Vector representation
    createdAt?: number;         // Timestamp
    similarity?: number;        // Used in search results
    shared?: boolean;           // Whether shared across agents
}
```

## Knowledge Types

Eliza supports different approaches to knowledge integration:

### 1. Static Character Knowledge

Basic knowledge defined directly in character configuration:

```json
{
  "name": "TechAdvisor",
  "knowledge": [
    "Python is a programming language created by Guido van Rossum.",
    "JavaScript is primarily used for web development.",
    "Machine Learning is a subset of artificial intelligence."
  ]
}
```

Pros:

- Simple to implement
- Always available to the character
- No additional processing required

Cons:

- Limited in size (token constraints)
- No semantic search capabilities
- Static content (requires redeployment to update)

### 2. RAG Knowledge

Dynamic knowledge stored in the database with semantic search:

```typescript
// Adding a knowledge document
await runtime.ragKnowledgeManager.processFile({
    path: "tech_guide.md",
    content: documentContent,
    type: "md",
    isShared: false
});
```

Pros:

- Supports large knowledge bases
- Dynamic updates without redeployment
- Advanced semantic search
- Context-aware retrieval

Cons:

- More complex to set up
- Requires vector database
- Additional processing overhead

### 3. Directory-based Knowledge

Structured knowledge files organized in directories:

```
/characters/tech_advisor/
  tech_advisor.json
  /knowledge/
    programming_languages.md
    cloud_computing.md
    machine_learning.md
    web_development.md
```

Pros:

- Organized file structure
- Easy to maintain and update
- Good for domain-specific knowledge

Cons:

- Requires proper file organization
- Manual loading process

## Setting Up Knowledge

### 1. Static Knowledge Setup

In the character configuration file:

```json
{
  "name": "FinanceExpert",
  "username": "finance-expert",
  "knowledge": [
    "Inflation is the rate at which the general level of prices rises.",
    "A bond is a fixed-income instrument representing a loan.",
    "Diversification is a risk management strategy."
  ]
}
```

### 2. RAG Knowledge Setup

#### Directory Structure

```
/characters/finance_expert/
  finance_expert.json
  /knowledge/
    investing.md
    taxation.md
    retirement.md
    market_analysis.md
```

#### Automatic Loading

Enable RAG in character file:

```json
{
  "name": "FinanceExpert",
  "username": "finance-expert",
  "ragKnowledge": true
}
```

#### Programmatic Loading

```typescript
// Processing individual files
await runtime.ragKnowledgeManager.processFile({
    path: "investing.md",
    content: fs.readFileSync("investing.md", "utf-8"),
    type: "md",
    isShared: false
});

// Processing multiple files
const knowledgeFiles = fs.readdirSync("knowledge");
for (const file of knowledgeFiles) {
    await runtime.ragKnowledgeManager.processFile({
        path: file,
        content: fs.readFileSync(`knowledge/${file}`, "utf-8"),
        type: file.endsWith(".md") ? "md" : "txt",
        isShared: false
    });
}
```

## Retrieval-Augmented Generation

RAG is the core technology for dynamic knowledge retrieval:

### 1. Document Processing

```typescript
// Process a document with chunking
const chunkSize = 512;  // Token size of each chunk
const bleed = 20;       // Overlap between chunks

await knowledge.set(runtime, {
    content: { text: documentContent },
    userId: "system",
    agentId: runtime.agentId
}, chunkSize, bleed);
```

### 2. Knowledge Retrieval

```typescript
// Basic retrieval
const knowledgeItems = await knowledge.get(runtime, userMessage);

// Advanced RAG retrieval
const relevantKnowledge = await runtime.ragKnowledgeManager.getKnowledge({
    query: userMessage,
    limit: 5,
    conversationContext: recentMessages
});
```

### 3. Context Integration

```typescript
// Format knowledge for context
const knowledgeContext = relevantKnowledge
    .map(item => item.content.text)
    .join("\n\n");

// Create combined context
const context = `
Relevant knowledge:
${knowledgeContext}

Conversation history:
${conversationHistory}
`;

// Generate response with context
const response = await generateText(runtime, context, userMessage);
```

## Knowledge Integration API

### Basic Knowledge API

```typescript
// Set knowledge item
await knowledge.set(
    runtime,                   // Agent runtime
    knowledgeItem,             // Item to store
    chunkSize = 512,           // Size of chunks (tokens)
    bleed = 20                 // Overlap between chunks
);

// Get knowledge based on query
const items = await knowledge.get(
    runtime,                   // Agent runtime
    query,                     // Query message
    options = {                // Optional parameters
        count: 5,              // Number of results
        threshold: 0.7         // Similarity threshold
    }
);

// Preprocess content
const cleaned = knowledge.preprocess(rawContent);
```

### RAG Knowledge API

```typescript
// Create knowledge
await ragKnowledgeManager.createKnowledge({
    id: uuid(),
    content: { text: "Knowledge content" },
    agentId: agentId,
    shared: false
});

// Get knowledge with advanced options
const knowledge = await ragKnowledgeManager.getKnowledge({
    query: "user query",                  // Search query
    conversationContext: recentMessages,  // Recent conversations
    limit: 5,                             // Result limit
    threshold: 0.85                       // Similarity threshold
});

// Process file
await ragKnowledgeManager.processFile({
    path: "document.md",              // File path
    content: documentContent,         // File content
    type: "md",                       // File type (md, txt, pdf)
    isShared: false                   // Visibility across agents
});

// Clear knowledge
await ragKnowledgeManager.clearKnowledge(shared = false);
```

### Embedding API

```typescript
// Generate embedding
const embedding = await embed(runtime, "Text to embed");

// Get embedding type
const embeddingType = getEmbeddingType(runtime);  // "local" or "remote"

// Get zero vector
const zeroVector = getEmbeddingZeroVector();

// Get embedding configuration
const config = getEmbeddingConfig();
```

## Provider-Knowledge Integration

The Eliza framework's providers and knowledge systems work together to create more intelligent and contextual agent responses.

### How Providers Access Knowledge

Providers can access the knowledge system in several ways:

```typescript
// Basic knowledge retrieval in a provider
const knowledgeProvider: Provider = {
    get: async (runtime: IAgentRuntime, message: Memory, state?: State) => {
        // Using the knowledge manager
        const knowledgeItems = await runtime.knowledgeManager.getRelevantKnowledge(
            message.content.text,
            5 // Limit to 5 most relevant chunks
        );
        
        // Using the RAG knowledge manager for more advanced retrieval
        const ragItems = await runtime.ragKnowledgeManager.getKnowledge({
            query: message.content.text,
            conversationContext: state.conversation,
            limit: 3,
            threshold: 0.85
        });
        
        // Combine and format for context
        return formatKnowledgeForContext(knowledgeItems.concat(ragItems));
    }
};
```

### Knowledge-Aware Providers

Creating providers that adapt based on available knowledge:

```typescript
const adaptiveProvider: Provider = {
    get: async (runtime: IAgentRuntime, message: Memory, state?: State) => {
        // Check if knowledge exists for this topic
        const knowledge = await runtime.knowledgeManager.getRelevantKnowledge(
            message.content.text,
            1
        );
        
        if (knowledge.length > 0) {
            // If knowledge exists, provide context for informed response
            return `You have information about this topic in your knowledge base.`;
        } else {
            // If no knowledge exists, provide context for more cautious response
            return `You don't have specific information about this topic in your knowledge base.`;
        }
    }
};
```

### Knowledge Updates and Provider Coordination

For optimal agent intelligence, coordinate knowledge updates with provider behavior:

1. **Knowledge Refresh Awareness**: Have providers adapt to newly added knowledge
2. **Knowledge Gap Detection**: Providers can identify and report gaps in knowledge
3. **Confidence Signaling**: Indicate confidence levels based on knowledge availability
4. **Cross-Referencing**: Compare requested information against available knowledge

For more details on provider implementation, refer to the `PROVIDER_COMMUNICATION.md` document.

## Randomization for Variety

Breaking knowledge into smaller chunks and selecting subsets creates more natural, varied responses. This is a key technique for building engaging agents.

### Benefits of Randomization

- **Prevents Repetition**: Avoids identical responses to similar queries
- **Creates Natural Variation**: Makes agent responses feel more human-like
- **Reduces Predictability**: Keeps interactions fresh and engaging
- **Covers Broader Context**: Allows for different aspects of knowledge to be highlighted

### Implementation Approaches

#### 1. Chunk-Level Randomization

```typescript
// Select a random subset of knowledge chunks
function getRandomizedKnowledge(allChunks, count = 3) {
    // Shuffle chunks
    const shuffled = [...allChunks].sort(() => Math.random() - 0.5);
    
    // Take top N chunks
    return shuffled.slice(0, count);
}

// Usage in knowledge retrieval
const relevantChunks = await knowledgeManager.getRelevantKnowledge(query);
const randomSubset = getRandomizedKnowledge(relevantChunks);
```

#### 2. Content-Level Randomization

```typescript
// Store multiple variations of the same knowledge
const knowledgeVariations = [
    "Python is a high-level programming language known for its readability.",
    "Python, created by Guido van Rossum, is valued for its clear syntax.",
    "Python emphasizes code readability with its notable use of whitespace."
];

// Select randomly when needed
function getRandomVariation(variations) {
    const index = Math.floor(Math.random() * variations.length);
    return variations[index];
}
```

#### 3. Progressive Disclosure

Gradually reveal different aspects of knowledge across interactions:

```typescript
// Track previously provided knowledge
const knowledgeHistory = new Set();

// Get novel knowledge items
async function getNovelKnowledge(runtime, query, count) {
    const allItems = await runtime.knowledgeManager.getRelevantKnowledge(query);
    
    // Filter out previously provided items
    const novelItems = allItems.filter(item => !knowledgeHistory.has(item.id));
    
    // If no novel items, reset history and start over
    if (novelItems.length === 0) {
        knowledgeHistory.clear();
        return allItems.slice(0, count);
    }
    
    // Select up to count novel items
    const selectedItems = novelItems.slice(0, count);
    
    // Record these as provided
    selectedItems.forEach(item => knowledgeHistory.add(item.id));
    
    return selectedItems;
}
```

### Integration with Character Files

Character files can specify randomization preferences:

```json
{
  "name": "Professor Bot",
  "knowledge": [...],
  "settings": {
    "knowledgeRandomization": {
      "enabled": true,
      "chunkCount": 3,
      "minimumRelevance": 0.7,
      "allowRepetition": false
    }
  }
}
```

## Configuration Options

### Environment Variables

```
# Embedding selection
USE_OPENAI_EMBEDDING=true
USE_OLLAMA_EMBEDDING=false
USE_GAIANET_EMBEDDING=false
USE_HEURIST_EMBEDDING=false
USE_BGE_EMBEDDING=false

# Database configuration
POSTGRES_URI=postgres://user:password@host:port/database
USE_SQLITE=false

# RAG parameters
RAG_MATCH_THRESHOLD=0.85
RAG_MATCH_COUNT=8
```

### Character Configuration

```json
{
  "name": "ExpertBot",
  "username": "expert-bot",
  "ragKnowledge": true,
  "knowledge": [
    "High-priority knowledge that should always be available."
  ]
}
```

### Runtime Options

```typescript
// RAG match parameters
const defaultRAGMatchThreshold = 0.85;  // Higher = more specific matches
const defaultRAGMatchCount = 8;         // Number of knowledge items to retrieve

// Chunking parameters
const chunkSize = 512;  // Size of each fragment in tokens
const bleed = 20;       // Overlap between fragments in tokens
```

## Best Practices

### Document Preparation

- **Break into logical sections**: Organize documents with clear headings
- **Clean content**: Remove irrelevant markup, code snippets, etc.
- **Descriptive titles**: Use clear filenames that indicate content
- **Format appropriately**: Use Markdown for structured content

### Knowledge Organization

- **Domain-specific files**: Create separate files for different knowledge domains
- **Hierarchical structure**: Organize by topics and subtopics
- **Context hints**: Include keywords that align with likely queries
- **Size optimization**: Keep chunks meaningful but not too large

### Retrieval Optimization

- **Tune thresholds**: Adjust match_threshold based on knowledge base size
- **Include context**: Add conversation context for better retrieval
- **Test with representative queries**: Validate retrieval with real examples
- **Balance quantity**: Adjust result limits to avoid context overflow

### Performance Considerations

- **Cache embeddings**: Reuse embeddings for common content
- **Batch processing**: Load documents in batches
- **Regular maintenance**: Clean up outdated knowledge
- **Monitor token usage**: Be aware of embedding and context token costs

## Advanced Techniques

### Knowledge Blending

Combining multiple knowledge sources:

```typescript
// Get knowledge from multiple sources
const ragItems = await runtime.ragKnowledgeManager.getKnowledge({
    query: userMessage,
    limit: 3
});

const staticKnowledge = character.knowledge || [];

const webSearchResults = await webSearch(userMessage, 2);

// Combine and prioritize
const combinedKnowledge = [
    ...staticKnowledge,
    ...ragItems.map(item => item.content.text),
    ...webSearchResults
];

// Create weighted context
const context = weightedCombine(combinedKnowledge);
```

### Contextual Reranking

Improving retrieval relevance:

```typescript
// Get initial knowledge matches
const initialMatches = await runtime.ragKnowledgeManager.searchKnowledge({
    query: userMessage,
    limit: 10
});

// Rerank based on conversation context
const reranked = rerank(initialMatches, {
    recentMessages,
    userPreferences,
    currentTopic
});

// Use top reranked items
const topItems = reranked.slice(0, 5);
```

### Knowledge Refreshing

Keeping knowledge up-to-date:

```typescript
// Check knowledge age
const knowledgeStats = await runtime.ragKnowledgeManager.getStats();

// Refresh if older than threshold
if (knowledgeStats.oldestItem < Date.now() - REFRESH_THRESHOLD) {
    // Clear old knowledge
    await runtime.ragKnowledgeManager.clearKnowledge();
    
    // Load fresh knowledge
    await loadKnowledgeBase("path/to/knowledge");
}
```

## Troubleshooting

### Common Issues

1. **Poor Retrieval Quality**
   - Check embedding quality and consistency
   - Adjust similarity threshold (lower for more results)
   - Ensure knowledge chunks are appropriately sized
   - Review preprocessing to ensure clean text

2. **Missing Knowledge**
   - Verify documents were processed successfully
   - Check agentId association is correct
   - Ensure shared flag is set appropriately
   - Confirm embedding dimension consistency

3. **Performance Issues**
   - Review database indexing for vector searches
   - Implement caching for frequent embeddings
   - Optimize chunk size for your specific use case
   - Consider batching for large knowledge bases

4. **Integration Problems**
   - Ensure knowledge is included in context correctly
   - Check knowledge format is understood by the model
   - Verify context length doesn't exceed model limits
   - Test knowledge retrieval independently of generation

### Debugging Tools

```typescript
// Test knowledge retrieval
const testQuery = "Test query";
const retrieved = await runtime.ragKnowledgeManager.getKnowledge({
    query: testQuery,
    limit: 5
});
console.log(`Retrieved ${retrieved.length} items for query "${testQuery}"`);

// Check embeddings
const embedding = await embed(runtime, "Test text");
console.log(`Embedding dimensions: ${embedding.length}`);

// Verify knowledge exists
const count = await runtime.ragKnowledgeManager.countKnowledge();
console.log(`Total knowledge items: ${count}`);
```

## Resources

- [Official Eliza Documentation](https://elizaos.github.io/eliza/docs/core/characterfile/)
- [Knowledge Management Tools](https://elizaos.github.io/eliza/docs/core/characterfile/)
- [Provider Communication Guide](./PROVIDER_COMMUNICATION.md)
- [Eliza GitHub Repository](https://github.com/elizaos/eliza/)
- [ElizaOS Community](https://elizaos.github.io/eliza)

---

By mastering the knowledge integration capabilities of the Eliza framework, you can create agents with deep domain expertise, accurate information retrieval, and dynamic knowledge access. The combination of static and RAG-based knowledge systems provides a flexible foundation for creating sophisticated AI applications.
