# Message Processing in Eliza Framework

This guide provides a comprehensive overview of message processing in the Eliza framework, explaining how messages flow through the system across different client platforms while maintaining consistent conversation context.

## Table of Contents

- [Core Concepts](#core-concepts)
- [Message Flow Architecture](#message-flow-architecture)
- [Client-Side Message Handling](#client-side-message-handling)
- [Server-Side Message Processing](#server-side-message-processing)
- [Conversation Context Management](#conversation-context-management)
- [Message Generation Pipeline](#message-generation-pipeline)
- [Action Processing](#action-processing)
- [Platform-Specific Message Handling](#platform-specific-message-handling)
- [Core APIs and Methods](#core-apis-and-methods)
- [Best Practices](#best-practices)
- [Advanced Techniques](#advanced-techniques)
- [Troubleshooting](#troubleshooting)

## Core Concepts

In the Eliza framework, message processing involves several key concepts:

- **Messages**: Units of communication between users and agents
- **Memories**: Stored message records with associated metadata and embeddings
- **Clients**: Platform-specific interfaces for user interaction
- **Context**: The comprehensive state used for response generation
- **Actions**: Operations that agents can perform beyond text responses
- **State**: The complete representation of conversation and agent information

## Message Flow Architecture

The message flow in Eliza follows this general pattern:

```
User Input → Client → Server → Memory Storage → Context Composition
    ↓                                               ↓
Client ← Response Processing ← Action Handling ← LLM Generation
```

This architecture enables:
1. Platform-independent core processing
2. Consistent memory management
3. Flexible client implementation
4. Extensible action system

## Client-Side Message Handling

### Web Client Implementation

The web client processes messages in the following way:

```typescript
// User sends a message
const handleSendMessage = async (message) => {
  // Update UI optimistically
  setMessages(prev => [...prev, { 
    text: message, 
    user: "user", 
    createdAt: Date.now() 
  }]);
  
  // Send to server
  const response = await api.sendMessage(agentId, message, selectedFile);
  
  // Update with agent response
  setMessages(prev => [...prev, {
    text: response.text,
    attachments: response.attachments,
    user: "agent",
    createdAt: Date.now()
  }]);
};
```

### API Client

The API client handles the communication with the server:

```typescript
// In client/src/lib/api.ts
sendMessage: (agentId, message, file = null) => {
  const formData = new FormData();
  formData.append("text", message);
  formData.append("user", "user");
  
  if (file) {
    formData.append("file", file);
  }
  
  return fetcher({
    url: `/${agentId}/message`,
    method: "POST",
    body: formData,
  });
}
```

## Server-Side Message Processing

### DirectClient Message Handling

The server processes incoming messages through the `DirectClient` class:

```typescript
app.post("/:agentId/message", upload.single("file"), async (req, res) => {
  // Extract parameters
  const agentId = req.params.agentId;
  const roomId = stringToUuid(req.body.roomId ?? "default-room-" + agentId);
  const userId = stringToUuid(req.body.userId ?? "user");
  
  // Get runtime for the agent
  const runtime = this.agents.get(agentId);
  
  // Create memory object
  const memory = {
    id: uuidv4(),
    agentId: runtime.agentId,
    userId,
    roomId,
    content: { text: req.body.text },
    createdAt: Date.now(),
  };
  
  // Process and store the message
  await runtime.processMessage(memory);
  
  // Generate and return response
  const response = await runtime.generateResponse(memory);
  res.json(response);
});
```

### Runtime Message Processing

The `AgentRuntime` class handles the core message processing:

```typescript
async processMessage(memory: Memory): Promise<void> {
  // Add embedding for semantic search
  const memoryWithEmbedding = await this.messageManager.addEmbeddingToMemory(memory);
  
  // Store in message history
  await this.messageManager.createMemory(memoryWithEmbedding);
  
  // Update user profile if needed
  await this.updateUserProfile(memory);
}

async generateResponse(memory: Memory): Promise<Memory> {
  // Compose the state with conversation context
  const state = await this.composeState(memory);
  
  // Generate response using LLM
  const response = await generateMessageResponse({
    runtime: this,
    context: this.formatContext(state),
    modelClass: ModelClass.LARGE,
  });
  
  // Create response memory
  const responseMemory = {
    id: uuidv4(),
    agentId: this.agentId,
    userId: this.agentId,
    roomId: memory.roomId,
    content: response,
    createdAt: Date.now(),
  };
  
  // Add embedding and store
  await this.messageManager.addEmbeddingToMemory(responseMemory);
  await this.messageManager.createMemory(responseMemory);
  
  // Process any actions in the response
  await this.processActions(memory, [responseMemory], state);
  
  return responseMemory;
}
```

## Conversation Context Management

Eliza uses a sophisticated memory system to maintain conversation context across interactions.

### Memory Manager

The `MemoryManager` class handles storing and retrieving messages:

```typescript
export class MemoryManager implements IMemoryManager {
  tableName: string;
  runtime: IAgentRuntime;
  
  constructor(runtime: IAgentRuntime, tableName: string) {
    this.runtime = runtime;
    this.tableName = tableName;
  }
  
  // Store memory in database
  async createMemory(memory: Memory, unique = false): Promise<void> {
    await this.runtime.databaseAdapter.createMemory(
      memory, 
      this.tableName, 
      unique
    );
  }
  
  // Get recent message history
  async getMemories(options: {
    roomId: UUID;
    count?: number;
    unique?: boolean;
    start?: number;
    end?: number;
  }): Promise<Memory[]> {
    return await this.runtime.databaseAdapter.getMemories({
      tableName: this.tableName,
      agentId: this.runtime.agentId,
      ...options,
    });
  }
  
  // Add embedding for semantic search
  async addEmbeddingToMemory(memory: Memory): Promise<Memory> {
    if (!memory.embedding && memory.content.text) {
      try {
        memory.embedding = await embed(this.runtime, memory.content.text);
      } catch (error) {
        memory.embedding = getEmbeddingZeroVector();
      }
    }
    return memory;
  }
  
  // Search memories by semantic similarity
  async searchMemoriesByEmbedding(
    embedding: number[],
    options: {
      roomId: UUID;
      match_threshold?: number;
      count?: number;
      unique?: boolean;
    }
  ): Promise<Memory[]> {
    return await this.runtime.databaseAdapter.searchMemories({
      tableName: this.tableName,
      agentId: this.runtime.agentId,
      embedding,
      match_threshold: options.match_threshold ?? 0.7,
      match_count: options.count ?? 10,
      roomId: options.roomId,
      unique: options.unique ?? false,
    });
  }
}
```

### State Composition

The `composeState` method creates a comprehensive context for response generation:

```typescript
async composeState(message: Memory): Promise<State> {
  // Initialize state
  const state: State = {
    agentName: this.character.name,
    conversation: [],
    goals: [],
    actions: [],
    bio: this.character.system,
  };
  
  // Get recent messages
  const recentMessages = await this.messageManager.getMemories({
    roomId: message.roomId,
    count: this.conversationLength,
  });
  
  // Format conversation
  state.conversation = formatMessages(recentMessages);
  
  // Get active goals
  state.goals = await this.getActiveGoals(message.roomId);
  
  // Format available actions
  state.actions = formatActions(this.actions);
  
  // Get relevant knowledge
  const knowledgeItems = await this.knowledgeManager.searchMemoriesByEmbedding(
    message.embedding,
    { roomId: "knowledge", count: 5 }
  );
  
  if (knowledgeItems.length > 0) {
    state.knowledge = knowledgeItems.map(k => k.content.text).join("\n\n");
  }
  
  // Get user profile
  const userDescriptions = await this.descriptionManager.getMemories({
    roomId: message.roomId,
    count: 1,
  });
  
  if (userDescriptions.length > 0) {
    state.userProfile = userDescriptions[0].content.text;
  }
  
  // Get character lore
  const lore = await this.loreManager.getMemories({
    roomId: "global",
    count: 5,
  });
  
  if (lore.length > 0) {
    state.lore = lore.map(l => l.content.text).join("\n\n");
  }
  
  // Add provider context
  const providerResults = await Promise.all(
    this.providers.map(provider => provider.get(this, message, state))
  );
  
  const providerContext = providerResults
    .filter(result => result != null && result !== "")
    .join("\n\n");
  
  if (providerContext) {
    state.providers = addHeader(
      `# Additional Information About ${this.character.name} and The World`,
      providerContext
    );
  }
  
  return state;
}
```

## Message Generation Pipeline

The message generation process utilizes the following steps:

### 1. Context Formatting

```typescript
formatContext(state: State): string {
  // Use template to format context
  return `
# Character: ${state.agentName}

## Bio
${state.bio}

${state.lore ? `## Background\n${state.lore}\n\n` : ''}

${state.userProfile ? `## User Profile\n${state.userProfile}\n\n` : ''}

${state.knowledge ? `## Relevant Knowledge\n${state.knowledge}\n\n` : ''}

${state.providers ? state.providers + '\n\n' : ''}

${state.goals.length > 0 ? `## Goals\n${formatGoalsAsString(state.goals)}\n\n` : ''}

${state.actions.length > 0 ? `## Available Actions\n${state.actions}\n\n` : ''}

## Conversation
${state.conversation}

${state.agentName}:`;
}
```

### 2. Message Generation

```typescript
async generateMessageResponse({
  runtime,
  context,
  modelClass = ModelClass.LARGE,
}): Promise<Content> {
  // Get model settings
  const modelSettings = getModelSettings(runtime, modelClass);
  
  // Generate text using the appropriate provider
  const textResponse = await generateText(runtime, context, {
    model: modelSettings.model,
    temperature: modelSettings.temperature,
    max_tokens: modelSettings.max_tokens,
    provider: runtime.modelProvider,
  });
  
  // Create response content
  return {
    text: textResponse,
  };
}
```

### 3. Action Detection and Processing

```typescript
async processActions(
  userMessage: Memory,
  responses: Memory[],
  state: State,
  callback?: (newMessages: Memory[]) => Promise<Memory[]>
): Promise<Memory[]> {
  // Check if the agent's response includes any actions
  for (const action of this.actions) {
    // Parse action from response text
    const actionResponse = parseActionResponseFromText(
      responses[0].content.text,
      action.name
    );
    
    if (actionResponse) {
      // Execute the action
      const result = await action.handler(
        userMessage,
        responses,
        state,
        this
      );
      
      // Update responses with action result
      if (result) {
        responses = result;
      }
      
      // Execute callback if provided
      if (callback) {
        responses = await callback(responses);
      }
      
      break; // Stop after first matching action
    }
  }
  
  return responses;
}
```

## Action Processing

Actions allow agents to perform operations beyond text responses:

### Action Definition

```typescript
// Example action definition
const generateImageAction: Action = {
  name: "generateImage",
  description: "Generate an image based on the given description",
  handler: async (userMessage, responses, state, runtime) => {
    // Extract image description from response
    const match = responses[0].content.text.match(/generateImage\("([^"]+)"\)/);
    if (!match) return responses;
    
    const imagePrompt = match[1];
    
    // Generate the image
    const imageUrl = await generateImage(runtime, {
      prompt: imagePrompt,
      width: 1024,
      height: 1024,
    });
    
    // Update the response to include the image
    const updatedResponse = {
      ...responses[0],
      content: {
        text: responses[0].content.text.replace(
          /generateImage\("[^"]+"\)/,
          `I've created an image for you:`
        ),
        attachments: [{
          type: "image",
          url: imageUrl,
        }],
      },
    };
    
    return [updatedResponse];
  },
};
```

### Action Registration

```typescript
// Register actions with the runtime
runtime.actions = [
  generateImageAction,
  searchWebAction,
  fetchWeatherAction,
  // other actions
];
```

## Platform-Specific Message Handling

Eliza supports multiple client platforms, each with specific message handling:

### Web Client Rendering

```tsx
// Rendering messages in the web client
<div className="message-container">
  {messages.map((message) => (
    <ChatBubble
      key={message.id}
      user={message.user === "user" ? "user" : "agent"}
      isLoading={message.isLoading}
    >
      {message.user === "agent" ? (
        <>
          <Markdown>{message.text}</Markdown>
          {message.attachments?.map((attachment) => (
            <div key={attachment.url} className="attachment">
              {attachment.type === "image" ? (
                <img src={attachment.url} alt="Generated" />
              ) : (
                <a href={attachment.url} target="_blank" rel="noreferrer">
                  {attachment.title || "Attachment"}
                </a>
              )}
            </div>
          ))}
        </>
      ) : (
        <div>{message.text}</div>
      )}
    </ChatBubble>
  ))}
</div>
```

### Discord Client

```typescript
// Discord-specific message handling
const handleDiscordMessage = async (message) => {
  // Ignore bot messages
  if (message.author.bot) return;
  
  // Create memory object
  const memory = {
    id: uuidv4(),
    agentId: runtime.agentId,
    userId: message.author.id,
    roomId: message.channel.id,
    content: { text: message.content },
    createdAt: Date.now(),
  };
  
  // Process message
  await runtime.processMessage(memory);
  
  // Generate response
  const response = await runtime.generateResponse(memory);
  
  // Handle Discord-specific formatting
  let replyContent = response.content.text;
  
  // Handle attachments
  if (response.content.attachments?.length > 0) {
    const files = response.content.attachments.map((attachment) => {
      return { attachment: attachment.url, name: attachment.title || "attachment" };
    });
    
    await message.reply({ content: replyContent, files });
  } else {
    // Split long messages if needed (Discord has 2000 char limit)
    if (replyContent.length > 2000) {
      const chunks = splitText(replyContent, 1900);
      for (const chunk of chunks) {
        await message.channel.send(chunk);
      }
    } else {
      await message.reply(replyContent);
    }
  }
};
```

### CLI Client

```typescript
// CLI-specific message handling
const handleCliMessage = async (input) => {
  // Create memory object
  const memory = {
    id: uuidv4(),
    agentId: runtime.agentId,
    userId: "cli-user",
    roomId: "cli-session",
    content: { text: input },
    createdAt: Date.now(),
  };
  
  // Process message
  await runtime.processMessage(memory);
  
  // Generate response
  const response = await runtime.generateResponse(memory);
  
  // Format CLI output
  console.log("\n🤖 " + chalk.cyan(runtime.character.name) + ":");
  console.log(response.content.text);
  
  // Handle any attachments
  if (response.content.attachments?.length > 0) {
    console.log("\nAttachments:");
    for (const attachment of response.content.attachments) {
      console.log(`- ${attachment.title || "Attachment"}: ${attachment.url}`);
    }
  }
};
```

## Core APIs and Methods

The key APIs for message processing include:

### MemoryManager API

```typescript
// Core memory operations
interface IMemoryManager {
  tableName: string;
  createMemory(memory: Memory, unique?: boolean): Promise<void>;
  getMemories(options: MemoryOptions): Promise<Memory[]>;
  getMemoryById(id: UUID): Promise<Memory | null>;
  addEmbeddingToMemory(memory: Memory): Promise<Memory>;
  searchMemoriesByEmbedding(
    embedding: number[],
    options: SearchOptions
  ): Promise<Memory[]>;
  removeMemory(id: UUID): Promise<void>;
  removeAllMemories(roomId: UUID): Promise<void>;
  countMemories(roomId: UUID): Promise<number>;
}
```

### AgentRuntime API

```typescript
// Core message processing methods
interface IAgentRuntime {
  processMessage(memory: Memory): Promise<void>;
  generateResponse(memory: Memory): Promise<Memory>;
  composeState(message: Memory, initialState?: Partial<State>): Promise<State>;
  processActions(
    userMessage: Memory,
    responses: Memory[],
    state: State,
    callback?: ActionCallback
  ): Promise<Memory[]>;
}
```

### Database Adapter API

```typescript
// Database operations for message storage
interface IDatabaseAdapter {
  createMemory(memory: Memory, tableName: string, unique?: boolean): Promise<void>;
  getMemories(options: DBMemoryOptions): Promise<Memory[]>;
  getMemoryById(id: UUID, tableName: string): Promise<Memory | null>;
  searchMemories(options: DBSearchOptions): Promise<Memory[]>;
  removeMemory(id: UUID, tableName: string): Promise<void>;
  removeAllMemories(roomId: UUID, tableName: string): Promise<void>;
  countMemories(roomId: UUID, tableName: string): Promise<number>;
}
```

## Best Practices

### Efficient Memory Management

- **Use embeddings** for all messages to enable semantic search
- **Set reasonable conversation lengths** to prevent context overflow
- **Implement memory cleanup** for old conversations
- **Store user profiles** separately from message history

### Optimal Message Processing

- **Initialize runtime with core components** before processing messages
- **Handle all message formats consistently** across platforms
- **Use structured content** with proper typing
- **Process attachments** consistently across platforms

### Effective Context Building

- **Include relevant knowledge** based on message content
- **Structure context clearly** with sections and headers
- **Limit context size** to work within model token limits
- **Prioritize recent messages** but include semantic search results

### Action Implementation

- **Design actions for specific use cases** with clear interfaces
- **Handle action errors gracefully** to maintain conversation flow
- **Update response content** to reflect action results
- **Use callback mechanisms** for stateful actions

## Advanced Techniques

### Message Chain Processing

For complex multi-step interactions:

```typescript
async processMessageChain(initialMessage: Memory, steps: number = 3): Promise<Memory[]> {
  let messages: Memory[] = [initialMessage];
  let currentMessage = initialMessage;
  
  for (let i = 0; i < steps; i++) {
    // Process the current message
    await this.processMessage(currentMessage);
    
    // Generate a response
    const response = await this.generateResponse(currentMessage);
    messages.push(response);
    
    // Use the response as the next input
    currentMessage = {
      id: uuidv4(),
      agentId: this.agentId,
      userId: initialMessage.userId,
      roomId: initialMessage.roomId,
      content: { 
        text: `Based on your last response, please continue with the next step.`
      },
      createdAt: Date.now(),
    };
  }
  
  return messages;
}
```

### Contextual Memory Retrieval

For more intelligent memory search:

```typescript
async getRelevantMemories(message: Memory): Promise<Memory[]> {
  // Get recent temporal context
  const recentMemories = await this.messageManager.getMemories({
    roomId: message.roomId,
    count: 5,
  });
  
  // Get semantically similar memories
  const similarMemories = await this.messageManager.searchMemoriesByEmbedding(
    message.embedding,
    {
      roomId: message.roomId,
      count: 5,
      match_threshold: 0.7,
    }
  );
  
  // Get user profile information
  const userProfiles = await this.descriptionManager.getMemories({
    roomId: message.roomId,
    count: 1,
  });
  
  // Combine and deduplicate
  const combinedMemories = [...recentMemories];
  
  for (const memory of similarMemories) {
    if (!combinedMemories.some(m => m.id === memory.id)) {
      combinedMemories.push(memory);
    }
  }
  
  return [...combinedMemories, ...userProfiles];
}
```

### Multi-Modal Response Generation

For responses with mixed media types:

```typescript
async generateMultiModalResponse(message: Memory): Promise<Memory> {
  // Generate text response
  const state = await this.composeState(message);
  const textResponse = await generateMessageResponse({
    runtime: this,
    context: this.formatContext(state),
  });
  
  // Detect if image generation is needed
  const shouldGenerateImage = textResponse.text.includes("[GENERATE_IMAGE]");
  
  if (shouldGenerateImage) {
    // Extract image prompt
    const imagePromptMatch = textResponse.text.match(/\[GENERATE_IMAGE: ([^\]]+)\]/);
    const imagePrompt = imagePromptMatch ? imagePromptMatch[1] : textResponse.text;
    
    // Generate image
    const imageUrl = await generateImage(this, {
      prompt: imagePrompt,
      width: 1024,
      height: 1024,
    });
    
    // Clean up response text
    const cleanText = textResponse.text.replace(/\[GENERATE_IMAGE: [^\]]+\]/, "");
    
    // Create response with text and image
    return {
      id: uuidv4(),
      agentId: this.agentId,
      userId: this.agentId,
      roomId: message.roomId,
      content: {
        text: cleanText,
        attachments: [{
          type: "image",
          url: imageUrl,
        }],
      },
      createdAt: Date.now(),
    };
  }
  
  // Return text-only response
  return {
    id: uuidv4(),
    agentId: this.agentId,
    userId: this.agentId,
    roomId: message.roomId,
    content: textResponse,
    createdAt: Date.now(),
  };
}
```

## Troubleshooting

### Common Issues

1. **Missing Context**
   - Check memory retrieval options
   - Verify message storage is working
   - Ensure embedding generation is functioning

2. **Platform-Specific Formatting Problems**
   - Inspect client-specific formatting logic
   - Check for platform limitations (e.g., message length)
   - Verify attachment handling

3. **Action Execution Failures**
   - Check action handler implementation
   - Ensure proper error handling
   - Verify action parsing logic

4. **Performance Issues**
   - Monitor memory usage
   - Optimize embedding generation
   - Consider caching frequent queries

### Debugging Tools

```typescript
// Message tracing middleware
const traceMessages = async (req, res, next) => {
  console.log(`[${new Date().toISOString()}] Incoming message:`, {
    agentId: req.params.agentId,
    roomId: req.body.roomId,
    userId: req.body.userId,
    text: req.body.text.substring(0, 100) + (req.body.text.length > 100 ? "..." : ""),
  });
  
  const originalSend = res.send;
  res.send = function(body) {
    console.log(`[${new Date().toISOString()}] Outgoing response:`, {
      body: typeof body === "string" ? body.substring(0, 100) + (body.length > 100 ? "..." : "") : "[Object]",
    });
    return originalSend.call(this, body);
  };
  
  next();
};

// Memory inspection utility
const inspectMemory = async (runtime, roomId) => {
  const count = await runtime.messageManager.countMemories(roomId);
  const memories = await runtime.messageManager.getMemories({
    roomId,
    count: 10,
  });
  
  console.log(`Room ${roomId} has ${count} messages. Last 10:`);
  for (const memory of memories) {
    console.log(`[${new Date(memory.createdAt).toISOString()}] ${memory.userId}: ${
      memory.content.text.substring(0, 50) + (memory.content.text.length > 50 ? "..." : "")
    }`);
  }
};
```

---

By mastering message processing in the Eliza framework, you can create sophisticated conversational agents that maintain context across different platforms, generate contextually relevant responses, and execute complex actions beyond basic text exchanges.