# Provider Communication in Eliza Framework

This guide provides a comprehensive overview of provider communication in the Eliza framework, focusing on how context providers supply dynamic information to agents during message generation.

## Table of Contents

- [Core Concepts](#core-concepts)
- [Provider Architecture](#provider-architecture)
- [Creating Custom Providers](#creating-custom-providers)
- [Provider Integration](#provider-integration)
- [Built-in Providers](#built-in-providers)
- [Advanced Provider Techniques](#advanced-provider-techniques)
- [Configuration Options](#configuration-options)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)
- [Complete Provider Example](#complete-provider-example)
- [Knowledge Management](#knowledge-management)

## Core Concepts

In the Eliza framework, "providers" are specialized components that:

- Supply contextual information to agents during message generation
- Inject dynamic data into the agent's state
- Run in parallel during context construction
- Extend agent capabilities without modifying the core message flow

Providers serve as the agent's "senses" for gathering real-time information beyond what's in the conversation or knowledge base.

## Provider Architecture

### Interface Definition

Providers implement a simple but powerful interface:

```typescript
export interface Provider {
    get: (
        runtime: IAgentRuntime,
        message: Memory,
        state?: State,
    ) => Promise<any>;
}
```

This interface consists of a single `get` method that:

- Receives the agent runtime, current message, and optional state
- Returns data to be included in the agent's context
- Can be asynchronous for API calls or database queries

### Registration in Runtime

Providers are registered in the `AgentRuntime` class:

```typescript
/**
 * Context providers used to provide context for message generation.
 */
providers: Provider[] = [];
```

### Provider Execution Flow

1. User sends a message to the agent
2. Agent runtime's `composeState` method is called
3. All registered providers' `get` methods are executed in parallel
4. Results are filtered (removing null/undefined values)
5. Non-empty results are combined with newlines
6. Combined output is added to state as `providers` property with a header
7. The state is used to construct the context for the LLM

## Creating Custom Providers

### Basic Provider Implementation

```typescript
const weatherProvider: Provider = {
    get: async (runtime, message, state) => {
        // Return relevant weather information based on the message content
        return "Current weather: 72°F and sunny";
    }
};
```

### Conditional Provider Output

Providers can decide whether to return information based on relevance:

```typescript
const timeProvider: Provider = {
    get: async (runtime, message, state) => {
        // Only include time if message asks about time
        if (message.content.text.toLowerCase().includes("time")) {
            return `Current time: ${new Date().toLocaleTimeString()}`;
        }
        return null; // Return null to exclude from context
    }
};
```

### Integrating External APIs

```typescript
const stockProvider: Provider = {
    get: async (runtime, message, state) => {
        // Extract ticker symbol from message
        const match = message.content.text.match(/stock price for (\w+)/i);
        if (!match) return null;
        
        const ticker = match[1];
        try {
            // Fetch stock data from API
            const apiKey = runtime.getSetting("STOCK_API_KEY");
            const response = await fetch(`https://api.example.com/stocks/${ticker}?key=${apiKey}`);
            const data = await response.json();
            
            return `${ticker} price: $${data.price} (${data.change}%)`;
        } catch (error) {
            console.error("Stock API error:", error);
            return null;
        }
    }
};
```

## Provider Integration

### Registering Providers

Providers can be registered through several mechanisms:

#### 1. During Agent Initialization

```typescript
const agent = new AgentRuntime({
    // other options
    providers: [timeProvider, weatherProvider]
});
```

#### 2. Via Plugin System

```typescript
const myPlugin: Plugin = {
    name: "EnvironmentPlugin",
    description: "Adds environmental awareness to agents",
    providers: [timeProvider, weatherProvider],
    // other plugin properties
};
```

#### 3. After Agent Initialization

```typescript
// Method exists but not explicitly defined in the viewed code
agent.registerContextProvider(stockProvider);
```

### State Composition in Runtime

The provider results are incorporated during state composition:

```typescript
// In runtime.ts, composeState method
async composeState() {
    // ... other state preparation

    // Gather provider context
    const providerResults = await Promise.all(
        this.providers.map(provider => provider.get(this, message, state))
    );
    
    // Filter out null/undefined results and combine
    const providerContext = providerResults
        .filter(result => result != null && result !== "")
        .join("\n\n");
    
    // Add to state with header
    state.providers = addHeader(
        `# Additional Information About ${this.character.name} and The World`,
        providerContext
    );

    // ... further state processing
}
```

## Built-in Providers

Eliza includes several built-in providers for common contextual needs:

### Time Provider

Supplies current date and time information:

```typescript
const timeProvider: Provider = {
    get: async () => {
        const now = new Date();
        return `The current date and time is ${now.toUTCString()}.`;
    }
};
```

### Facts Provider

Retrieves relevant facts from memory:

```typescript
const factsProvider: Provider = {
    get: async (runtime, message) => {
        // Get relevant facts from memory
        const facts = await runtime.factsManager.getMemories({
            roomId: message.roomId,
            count: 5
        });
        
        if (facts.length === 0) return null;
        
        return `Important Facts:\n${facts.map(f => 
            `- ${f.content.text}`
        ).join('\n')}`;
    }
};
```

### Boredom Provider

Models agent engagement level in conversation:

```typescript
const boredomProvider: Provider = {
    get: async (runtime, message, state) => {
        // Calculate boredom level based on conversation patterns
        const boredomScore = calculateBoredom(state.conversation);
        if (boredomScore > 0.7) {
            return "You're feeling a bit bored with this conversation.";
        }
        return null;
    }
};
```

## Advanced Provider Techniques

### Dynamic Relevance Scoring

```typescript
const newsProvider: Provider = {
    get: async (runtime, message, state) => {
        // Generate embedding for the message
        const embedding = await embed(runtime, message.content.text);
        
        // Get latest news articles
        const articles = await fetchLatestNews();
        
        // Generate embeddings for article headlines
        const articleEmbeddings = await Promise.all(
            articles.map(a => embed(runtime, a.headline))
        );
        
        // Find most relevant articles based on cosine similarity
        const relevantArticles = articles.filter((article, i) => {
            const similarity = cosineSimilarity(embedding, articleEmbeddings[i]);
            return similarity > 0.7; // Only include highly relevant articles
        });
        
        if (relevantArticles.length === 0) return null;
        
        // Format relevant news for context
        return `Relevant News:\n${relevantArticles.map(a => 
            `- ${a.headline}: ${a.summary}`
        ).join('\n')}`;
    }
};
```

### Context-Aware Information Density

```typescript
const knowledgeProvider: Provider = {
    get: async (runtime, message, state) => {
        // Analyze current conversation complexity
        const complexity = analyzeComplexity(state.conversation);
        
        // Adjust information detail based on conversation complexity
        const detail = complexity > 0.7 ? "detailed" : "simple";
        
        // Get relevant knowledge with appropriate detail level
        const knowledge = await fetchKnowledge(message.content.text, detail);
        return knowledge;
    }
};
```

### Provider Chaining

```typescript
const combinedProvider: Provider = {
    get: async (runtime, message, state) => {
        // Run initial provider to get basic info
        const baseProvider = new BaseProvider();
        const baseInfo = await baseProvider.get(runtime, message, state);
        
        if (!baseInfo) return null;
        
        // Use that info to get enhanced data
        const enhancedProvider = new EnhancedProvider(baseInfo);
        const enhancedInfo = await enhancedProvider.get(runtime, message, state);
        
        // Combine for richer context
        return `${baseInfo}\n\nAdditional Details:\n${enhancedInfo}`;
    }
};
```

## Configuration Options

Providers can be configured in several ways:

### 1. Environment Variables

Access via `process.env` or through the runtime's settings:

```typescript
const apiKey = runtime.getSetting("WEATHER_API_KEY");
```

### 2. Character Settings

Access character-specific configuration:

```typescript
const providerEnabled = runtime.character.settings?.enableWeatherProvider === true;
if (!providerEnabled) return null;
```

### 3. Runtime Options

Use runtime properties for configuration:

```typescript
const maxResults = runtime.maxProviderResults || 5;
```

### 4. Provider-specific Configuration

Pass configuration during provider creation:

```typescript
const createConfiguredProvider = (config) => ({
    get: async (runtime, message, state) => {
        // Use config in provider logic
        if (config.detailLevel === "high") {
            return getDetailedInfo();
        } else {
            return getBasicInfo();
        }
    }
});

// Usage
const myProvider = createConfiguredProvider({ detailLevel: "high" });
```

## Best Practices

### Performance Optimization

- **Be concise**: Provider output adds to token usage
- **Cache results**: Avoid redundant expensive operations
- **Use embeddings efficiently**: Cache embeddings when possible
- **Implement timeouts**: Prevent slow providers from delaying responses

### Error Handling

- **Graceful failures**: Return null on errors instead of throwing exceptions
- **Logging**: Log errors for debugging without breaking the response flow
- **Fallbacks**: Provide default information when external services fail

### Content Formatting

- **Clear structure**: Use consistent formatting for readability
- **HTML/Markdown considerations**: Format for the target model's expectations
- **Sectioning**: Use headers and bullets for better organization
- **Length management**: Keep output concise and relevant

### Context Relevance

- **Filter by topic**: Only return information relevant to the conversation
- **Conditional return**: Return null when information isn't needed
- **Prioritize importance**: Focus on the most critical information first
- **Consider recency**: Prioritize recent and timely information

## Troubleshooting

### Common Issues

1. **Provider Output Not Appearing**
   - Check if provider is returning null or empty string
   - Verify provider is properly registered
   - Ensure provider doesn't throw unhandled exceptions

2. **Performance Problems**
   - Identify slow providers using timing logs
   - Implement caching for expensive operations
   - Consider moving to asynchronous updates for slow data sources

3. **Inconsistent Results**
   - Check for race conditions in asynchronous code
   - Verify API stability for external data sources
   - Add logging to track provider execution

4. **Context Overflow**
   - Limit provider output length
   - Implement relevance filtering
   - Prioritize essential information

### Debugging Techniques

```typescript
// Add debugging to providers
const debuggableProvider: Provider = {
    get: async (runtime, message, state) => {
        console.time("myProvider execution");
        try {
            const result = await getInformation();
            console.log("Provider result:", result?.substring(0, 100) + "...");
            return result;
        } catch (error) {
            console.error("Provider error:", error);
            return null;
        } finally {
            console.timeEnd("myProvider execution");
        }
    }
};
```

## Complete Provider Example

Here's a full example of a weather provider implementation:

```typescript
import type { IAgentRuntime, Memory, Provider, State } from "@elizaos/core";

interface WeatherData {
    temperature: number;
    condition: string;
    location: string;
    forecast: string[];
}

const weatherProvider: Provider = {
    get: async (runtime: IAgentRuntime, message: Memory, state?: State) => {
        // Check if weather info is relevant to the conversation
        const messageText = message.content.text.toLowerCase();
        if (!messageText.includes("weather") && 
            !messageText.includes("temperature") && 
            !messageText.includes("forecast") &&
            !messageText.includes("outside")) {
            return null; // Skip if not relevant
        }
        
        try {
            // Extract location from message or use default
            const locationMatch = messageText.match(/weather (?:in|at|for) ([a-z ]+)/i);
            const location = locationMatch ? locationMatch[1].trim() : "the current location";
            
            // Check cache first to avoid redundant API calls
            const cacheKey = `weather:${location}:${new Date().toISOString().split('T')[0]}`;
            const cachedData = await runtime.cacheManager?.get(cacheKey);
            
            let weatherData: WeatherData;
            
            if (cachedData) {
                weatherData = JSON.parse(cachedData);
            } else {
                // Fetch weather data from API
                const apiKey = runtime.getSetting("WEATHER_API_KEY");
                if (!apiKey) {
                    return "Weather information is unavailable.";
                }
                
                const url = `https://api.weather.com/data?location=${encodeURIComponent(location)}&key=${apiKey}`;
                const response = await fetch(url);
                
                if (!response.ok) {
                    throw new Error(`Weather API error: ${response.status}`);
                }
                
                weatherData = await response.json() as WeatherData;
                
                // Cache the result for 1 hour
                await runtime.cacheManager?.set(cacheKey, JSON.stringify(weatherData), 60 * 60);
            }
            
            // Format response based on message intent
            if (messageText.includes("forecast")) {
                return `Weather forecast for ${weatherData.location}:
                - Current: ${weatherData.temperature}°C, ${weatherData.condition}
                - Forecast: ${weatherData.forecast.join("\n  ")}`;
            } else {
                return `Current weather in ${weatherData.location}: ${weatherData.temperature}°C, ${weatherData.condition}.`;
            }
        } catch (error) {
            console.error("Weather provider error:", error);
            // Return null instead of error message to avoid confusing the model
            return null;
        }
    }
};

export { weatherProvider };
```

## Knowledge Management

Knowledge integration is an important aspect of provider functionality. While comprehensive details are available in the dedicated `KNOWLEDGE_INTEGRATION.md` document, here's how providers interact with the knowledge system:

### Knowledge Access in Providers

Providers can access knowledge through the knowledge management systems:

```typescript
const knowledgeProvider: Provider = {
    get: async (runtime: IAgentRuntime, message: Memory, state?: State) => {
        // Access knowledge using the knowledge manager
        const relevantKnowledge = await runtime.knowledgeManager.getRelevantKnowledge(
            message.content.text,
            5 // Limit to 5 most relevant chunks
        );
        
        if (!relevantKnowledge.length) return null;
        
        // Format knowledge for inclusion in context
        return `Relevant Information:\n${relevantKnowledge.map(k => 
            `- ${k.content.text}`
        ).join('\n')}`;
    }
};
```

### Knowledge Tools Reference

Eliza provides specialized tools for knowledge management that providers can utilize:

- **folder2knowledge**: Converts a folder of documents into a knowledge file
- **knowledge2character**: Adds knowledge to a character file
- **tweets2character**: Imports tweets for a character's knowledge base

For detailed usage of these tools and comprehensive knowledge integration practices, refer to the `KNOWLEDGE_INTEGRATION.md` document.

### Knowledge-Provider Integration

When designing providers that interact with knowledge:

1. **Relevance Filtering**: Only include knowledge directly relevant to the current conversation
2. **Format Appropriately**: Present knowledge in a digestible format that fits the agent's style
3. **Prioritize Critical Information**: Present the most important knowledge first
4. **Reference Sources**: When appropriate, indicate where knowledge came from
5. **Coordinate with Knowledge Updates**: Design providers to work with the knowledge refresh cycle

---

By mastering provider communication in the Eliza framework, you can create agents with real-time awareness, dynamic information access, and contextually rich responses. The provider architecture enables you to extend agent capabilities without modifying core functionality, making it a powerful tool for enhancing agent intelligence and utility.
