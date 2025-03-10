# Agent Memory Management in Eliza Framework

This guide provides a comprehensive overview of how memory works in the Eliza agent framework and how to effectively use it to create agents with robust memory capabilities.

## Table of Contents

- [Core Concepts](#core-concepts)
- [Memory Architecture](#memory-architecture)
- [Memory Types](#memory-types)
- [Working with Memory](#working-with-memory)
- [Vector Embeddings and Semantic Search](#vector-embeddings-and-semantic-search)
- [Memory Configuration](#memory-configuration)
- [Best Practices](#best-practices)
- [Advanced Techniques](#advanced-techniques)
- [Troubleshooting](#troubleshooting)

## Core Concepts

In the Eliza framework, "memory" refers to an agent's ability to store, recall, and utilize information across conversations and interactions. This is fundamentally different from traditional programming memory management.

Key concepts:

- **Memories**: Discrete units of information stored by agents
- **Embedding**: Vector representations of text enabling semantic search
- **Retrieval**: Process of finding relevant memories based on context
- **Persistence**: Long-term storage of memories across sessions

## Memory Architecture

### Core Components

The memory system consists of several key components:

1. **MemoryManager**: Central component that handles all memory operations
2. **Database Adapters**: Abstract storage operations for different backends
3. **Embedding System**: Converts text to vector representations
4. **RAGKnowledgeManager**: Specialized manager for knowledge retrieval

### Implementation Flow

```
User Message → Memory Creation → Embedding Generation → Database Storage
                                                      ↓
Agent Response ← Context Assembly ← Memory Retrieval ← Query Embedding
```

### Memory Structure

Each memory record includes:

```typescript
interface Memory {
    id?: UUID;              // Unique identifier
    userId: UUID;           // User associated with memory
    agentId: UUID;          // Agent associated with memory
    createdAt?: number;     // Creation timestamp
    content: Content;       // Main content (text, media, etc.)
    embedding?: number[];   // Vector representation
    roomId: UUID;           // Conversation context
    unique?: boolean;       // Whether this is a unique memory
    similarity?: number;    // Used in search results
}
```

## Memory Types

Eliza provides specialized memory managers for different types of data:

### 1. Message Memory (`messageManager`)

Stores conversation history between users and agents.

```typescript
// Storing a message
await runtime.messageManager.createMemory({
    content: { text: "Hello, how can I help you today?" },
    userId: userId,
    agentId: agentId,
    roomId: roomId
});
```

### 2. Description Memory (`descriptionManager`)

Maintains persistent information about users.

```typescript
// Creating a user description
await runtime.descriptionManager.createMemory({
    content: { text: "John is a software engineer who likes hiking." },
    userId: userId,
    agentId: agentId,
    roomId: roomId
});
```

### 3. Lore Memory (`loreManager`)

Stores character background information and personality traits.

```typescript
// Adding character lore
await runtime.loreManager.createMemory({
    content: { text: "Agent Shaw is a cybersecurity expert with military background." },
    userId: "system",
    agentId: agentId,
    roomId: "global"
});
```

### 4. Document Memory (`documentsManager`)

Handles larger reference materials and documents.

```typescript
// Storing a document
await runtime.documentsManager.createMemory({
    content: { text: documentContent, metadata: { title: "User Manual" } },
    userId: "system",
    agentId: agentId,
    roomId: "global"
});
```

### 5. Knowledge Memory (`knowledgeManager`)

Contains searchable document fragments for quick reference.

```typescript
// Adding knowledge fragment
await runtime.knowledgeManager.createMemory({
    content: { text: "The product warranty expires after 12 months." },
    userId: "system",
    agentId: agentId,
    roomId: "knowledge"
});
```

### 6. RAG Knowledge (`ragKnowledgeManager`)

Specialized system for Retrieval-Augmented Generation.

```typescript
// Processing a knowledge file
await runtime.ragKnowledgeManager.processFile({
    path: "product-info.md",
    content: markdownContent,
    type: "md",
    isShared: false
});
```

## Working with Memory

### Creating Memories

```typescript
// Basic memory creation
await memoryManager.createMemory({
    id: uuidv4(),                            // Optional, auto-generated if omitted
    content: { text: "Memory content" },     // Required
    userId: userId,                          // Required
    agentId: agentId,                        // Required
    roomId: roomId                           // Required
});
```

### Retrieving Memories

```typescript
// Get recent memories
const memories = await memoryManager.getMemories({
    roomId: roomId,        // Conversation context
    count: 10,             // Number of memories to retrieve
    unique: true           // Exclude duplicates
});

// Format for context
const formattedContext = formatMessages(memories);
```

### Semantic Search

```typescript
// Generate embedding for query
const embedding = await embed(runtime, "search query");

// Search for similar memories
const results = await memoryManager.searchMemoriesByEmbedding(embedding, {
    roomId: roomId,
    match_threshold: 0.7,  // Similarity threshold (0-1)
    count: 5               // Maximum results
});
```

### Memory Management

```typescript
// Remove specific memory
await memoryManager.removeMemory(memoryId);

// Clear all memories in a room
await memoryManager.removeAllMemories(roomId);

// Count memories
const count = await memoryManager.countMemories(roomId);
```

## Vector Embeddings and Semantic Search

Eliza uses vector embeddings to enable semantic search capabilities.

### Embedding Generation

```typescript
// Create embedding vector
const embedding = await embed(runtime, "Text to embed");
```

Supported embedding providers:

- **OpenAI**: 1536 dimensions (default)
- **Local BGE models**: 384 dimensions
- **Ollama**: Various dimensions based on model
- **Other providers**: GaiaNet, Heurist, etc.

### Semantic Search Process

1. Convert query to embedding vector
2. Compare against stored memory embeddings using cosine similarity
3. Return memories above similarity threshold
4. Optionally rerank results based on additional criteria

### RAG Knowledge Enhancement

The RAG system enhances retrieval through:

- Term frequency matching
- Proximity analysis
- Chunk-based retrieval
- Stop word filtering
- Context-aware reranking

```typescript
// Advanced knowledge retrieval
const relevantKnowledge = await runtime.ragKnowledgeManager.getKnowledge({
    query: "How do I reset my password?",
    conversationContext: recentMessages,
    limit: 3
});
```

## Memory Configuration

### Database Adapters

```typescript
// SQLite adapter (development)
const sqliteAdapter = new SqliteDatabaseAdapter('path/to/database.db');

// PostgreSQL adapter (production)
const pgAdapter = new PostgresAdapter({
    connectionString: process.env.POSTGRES_URI
});
```

### Environment Variables

```
# Embedding configuration
USE_OPENAI_EMBEDDING=TRUE
OPENAI_API_KEY=sk-xxx

# Database configuration
POSTGRES_URI=postgres://user:password@host:port/database
SQLITE_PATH=./database.sqlite
```

### Memory Parameters

```typescript
// Default search parameters
const defaultMatchThreshold = 0.1;  // Minimum similarity score
const defaultMatchCount = 10;       // Default number of results
```

## Best Practices

### Memory Organization

- **Room-based Isolation**: Use separate roomIds for different contexts
- **User-specific Knowledge**: Tie memories to specific users when appropriate
- **Global Knowledge**: Use special roomIds like "global" or "knowledge" for shared information

### Performance Optimization

- **Embedding Caching**: Cache embeddings for frequently used content
- **Regular Cleanup**: Remove outdated or unnecessary memories
- **Indexing**: Ensure proper database indexing for vector operations

### Memory Quality

- **Uniqueness Checks**: Set `unique: true` to avoid duplicate memories
- **Content Preprocessing**: Clean and normalize text before embedding
- **Threshold Tuning**: Adjust match_threshold based on use case needs

## Advanced Techniques

### Custom Memory Managers

```typescript
class CustomMemoryManager extends MemoryManager {
    constructor(adapter, options) {
        super(adapter, options);
    }
    
    // Override methods as needed
    async searchMemoriesByEmbedding(embedding, options) {
        // Custom implementation
    }
}

// Register with runtime
runtime.registerMemoryManager('custom', customMemoryManager);
```

### Memory Blending

```typescript
// Retrieve from multiple sources
const messageMemories = await runtime.messageManager.getMemories(options);
const knowledgeMemories = await runtime.knowledgeManager.searchMemoriesByEmbedding(embedding, options);
const loreMemories = await runtime.loreManager.getMemories(options);

// Combine and rerank
const allMemories = [...messageMemories, ...knowledgeMemories, ...loreMemories];
const rerankedMemories = rerank(allMemories, query);
```

### Memory Reflection

Implement periodic review of memories to generate summaries or insights:

```typescript
// Get all memories for a user
const allUserMemories = await runtime.messageManager.getMemories({
    userId: userId,
    count: 100
});

// Generate summary or insights
const userProfile = await generateText(runtime, `
Based on these interactions, create a summary of this user:
${allUserMemories.map(m => m.content.text).join('\n')}
`);

// Store as description
await runtime.descriptionManager.createMemory({
    content: { text: userProfile },
    userId: userId,
    agentId: agentId,
    roomId: "profiles"
});
```

## Troubleshooting

### Common Issues

1. **Memory Not Found**
   - Check roomId, userId, and agentId parameters
   - Verify memory exists in database

2. **Poor Semantic Search Results**
   - Adjust match_threshold (lower for more results)
   - Check embedding model consistency
   - Review query formulation

3. **Performance Issues**
   - Implement database indexing
   - Add caching for frequent operations
   - Consider memory pruning for large datasets

4. **Embedding Dimension Mismatch**
   - Ensure consistent embedding providers
   - Verify vector dimensions match database expectations

### Debugging Tools

```typescript
// Memory inspection
const memoryCount = await memoryManager.countMemories(roomId);
console.log(`Memory count for room ${roomId}: ${memoryCount}`);

// Embedding verification
const testEmbedding = await embed(runtime, "Test text");
console.log(`Embedding dimensions: ${testEmbedding.length}`);

// Database validation
const dbStatus = await adapter.ping();
console.log(`Database status: ${dbStatus ? 'connected' : 'disconnected'}`);
```

---

By mastering the memory management capabilities of the Eliza framework, you can create agents with rich contextual awareness, personalized interactions, and powerful knowledge retrieval abilities. The combination of structured memory types, semantic search, and flexible storage options provides a robust foundation for advanced AI agent applications.