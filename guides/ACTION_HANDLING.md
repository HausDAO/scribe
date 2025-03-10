# Action Handling in Eliza Framework

This guide provides a comprehensive overview of action handling in the Eliza framework, explaining how agents can perform operations beyond simple message responses.

## Table of Contents

- [Core Concepts](#core-concepts)
- [Action Architecture](#action-architecture)
- [Creating Custom Actions](#creating-custom-actions)
- [Action Detection and Execution](#action-detection-and-execution)
- [Built-in Actions](#built-in-actions)
- [Advanced Action Techniques](#advanced-action-techniques)
- [Security and Validation](#security-and-validation)
- [Best Practices](#best-practices)
- [Testing Actions](#testing-actions)
- [Troubleshooting](#troubleshooting)

## Core Concepts

In the Eliza framework, actions are specialized operations that agents can perform in response to user messages:

- **Actions**: Defined operations that extend an agent's capabilities beyond text responses
- **Handlers**: Functions that implement action behavior
- **Validation**: Logic that determines when an action is appropriate
- **Examples**: Demonstrations that teach the model when to use actions
- **Action Flow**: The process from detection to execution

Actions transform agents from simple conversational entities into capable assistants that can interact with external systems, maintain state, and perform complex tasks.

## Action Architecture

### Action Interface

Actions in Eliza are defined through the `Action` interface:

```typescript
interface Action {
    // Unique identifier for the action
    name: string;
    
    // Alternative names/phrases that can trigger the action
    similes: string[];
    
    // Description of the action's purpose and usage
    description: string;
    
    // Example conversations showing when/how to use the action
    examples: ActionExample[][];
    
    // Function implementing the action's behavior
    handler: Handler;
    
    // Function determining if the action is appropriate
    validate: Validator;
    
    // Optional flag to suppress initial message when using this action
    suppressInitialMessage?: boolean;
}
```

#### Understanding `suppressInitialMessage`

The `suppressInitialMessage` property is a powerful flag that controls whether the initial text message is sent to the user before the action is executed:

- When `true`: The system will not send the text portion of the response before executing the action. This is useful for actions that need to replace the text with an updated response based on their execution.
- When `false` (default): The system sends the text message first, then executes the action. This provides immediate feedback to the user while a potentially time-consuming action runs.

Use cases for setting `suppressInitialMessage: true`:

- Actions that might fail and need to show a different message
- Actions that calculate information to be displayed in the response
- Actions where the initial message might confuse users before the action completes

### Handler and Validator Types

The core function types that power actions:

```typescript
// Handler function for implementing action behavior
type Handler = (
    runtime: IAgentRuntime,       // Agent runtime
    message: Memory,              // User message
    state?: State,                // Current state
    options?: { [key: string]: unknown }, // Optional parameters
    callback?: HandlerCallback    // Optional callback function
) => Promise<unknown>;

// Validator function for determining if action is appropriate
type Validator = (
    runtime: IAgentRuntime,       // Agent runtime
    message: Memory,              // User message 
    state?: State                 // Current state
) => Promise<boolean>;            // Return true if action is valid

// Callback for returning action results
type HandlerCallback = (response: Content) => Promise<void>;
```

### Action Examples

Examples teach the model when and how to use actions:

```typescript
type ActionExample = {
    // User who sent the message
    user: string;
    
    // Content of the message
    content: Content;
};
```

A typical example set:

```typescript
examples: [
    [
        { user: "{{user1}}", content: { text: "Can you transfer 50 tokens to @john?" } },
        { user: "{{user2}}", content: { text: "I'll transfer those tokens for you.", action: "TRANSFER_TOKENS" } }
    ],
    [
        { user: "{{user1}}", content: { text: "Send 20 SOL to this address: abc123..." } },
        { user: "{{user2}}", content: { text: "Processing your token transfer.", action: "TRANSFER_TOKENS" } }
    ]
]
```

#### Designing Effective Examples

Examples are crucial for teaching the model when to use your action. Following these patterns ensures reliable action detection:

1. **Cover Multiple Variations**: Include different phrasings and contexts
2. **Show Clear Triggers**: Make it obvious what user inputs should trigger the action
3. **Demonstrate Response Style**: Show the appropriate response tone and information to include
4. **Include Edge Cases**: Show how to handle ambiguous or partial requests

## Creating Custom Actions

### Basic Action Implementation

Here's how to create a custom action:

```typescript
import { Action, IAgentRuntime, Memory, State, HandlerCallback } from "@elizaos/core";

export const weatherAction: Action = {
    name: "WEATHER",
    similes: ["GET_WEATHER", "CHECK_WEATHER", "WEATHER_FORECAST"],
    description: "Gets the current weather or forecast for a specified location",
    
    validate: async (runtime: IAgentRuntime, message: Memory) => {
        // Check if this message is about weather
        const text = message.content.text.toLowerCase();
        return text.includes("weather") || 
               text.includes("temperature") || 
               text.includes("forecast");
    },
    
    handler: async (
        runtime: IAgentRuntime,
        message: Memory,
        state?: State,
        options?: any,
        callback?: HandlerCallback
    ) => {
        try {
            // Extract location from message
            const locationMatch = message.content.text.match(/weather (?:in|at|for) ([a-zA-Z, ]+)/i);
            const location = locationMatch ? locationMatch[1] : "current location";
            
            // Get weather data (mock implementation)
            const weatherData = await getWeatherData(location);
            
            // Prepare response
            const response = {
                text: `The current weather in ${location} is ${weatherData.temperature}°C with ${weatherData.condition}. The forecast for today shows ${weatherData.forecast}.`,
                action: null  // Clear the action
            };
            
            // Send response via callback
            if (callback) {
                await callback(response);
            }
            
            return true;
        } catch (error) {
            // Handle errors
            const errorResponse = {
                text: `I couldn't retrieve the weather information. ${error.message}`,
                action: null
            };
            
            if (callback) {
                await callback(errorResponse);
            }
            
            return false;
        }
    },
    
    examples: [
        [
            { user: "{{user1}}", content: { text: "What's the weather in New York?" } },
            { user: "{{user2}}", content: { text: "Let me check the weather for you.", action: "WEATHER" } }
        ],
        [
            { user: "{{user1}}", content: { text: "Will it rain tomorrow in London?" } },
            { user: "{{user2}}", content: { text: "I'll get the forecast for London.", action: "WEATHER" } }
        ]
    ]
};
```

### Action Registration

Register your action with the runtime:

```typescript
// Direct registration
runtime.registerAction(weatherAction);

// Via plugin
const weatherPlugin = {
    name: "weather-plugin",
    description: "Provides weather information",
    actions: [weatherAction]
};

runtime.registerPlugin(weatherPlugin);
```

Or specify in your character configuration:

```json
{
  "name": "WeatherBot",
  "plugins": ["@elizaos/plugin-weather"]
}
```

## Action Detection and Execution

### Detection Flow

The process of detecting and executing actions follows these steps:

1. **Agent Response Generation**: The agent generates a response that may include an action

   ```typescript
   // Example agent response
   {
       text: "I'll check the weather for you right away.",
       action: "WEATHER"
   }
   ```

2. **Action Detection**: The system identifies the action in the response

   ```typescript
   // Normalize action name
   const normalizedAction = response.content.action.toLowerCase().replace("_", "");
   
   // Find matching action
   const action = this.actions.find(a => 
       a.name.toLowerCase().replace("_", "").includes(normalizedAction) ||
       normalizedAction.includes(a.name.toLowerCase().replace("_", ""))
   );
   ```

3. **Validation**: The system validates that the action is appropriate

   ```typescript
   // Validate action is appropriate
   const isValid = await action.validate(this, message, state);
   if (!isValid) return responses;
   ```

4. **Execution**: The action's handler is executed

   ```typescript
   // Execute action handler
   const result = await action.handler(
       this,           // runtime
       message,        // original user message
       state,          // current state
       {},             // options
       async (newResponse) => {
           // Handle response updates
       }
   );
   ```

5. **Response Handling**: The action's response is processed

   ```typescript
   // Update with action result
   if (result && typeof result === "object") {
       responses = Array.isArray(result) ? result : [result];
   }
   ```

### Runtime Integration

Actions are fully integrated with the agent runtime:

```typescript
// In AgentRuntime class
async processActions(
    message: Memory,               // Original user message
    responses: Memory[],           // Current responses
    state: State,                  // Current state
    callback?: ActionCallback      // Optional callback
): Promise<Memory[]> {
    const response = responses[0];
    
    if (!response.content.action) {
        return responses;
    }
    
    // Find and execute matching action
    // ...implementation details...
    
    return responses;
}
```

## Built-in Actions

Eliza includes several built-in actions in the bootstrap plugin:

### CONTINUE Action

Allows the agent to continue its response without waiting for user input:

```typescript
const continueAction: Action = {
    name: "CONTINUE",
    similes: ["ELABORATE", "KEEP_TALKING"],
    description: "ONLY use this action when the message necessitates a follow up...",
    
    validate: async (runtime, message) => {
        // Prevent too many consecutive continuations
        const recentMessages = await runtime.messageManager.getMemories({
            roomId: message.roomId,
            count: 10
        });
        
        const consecutiveContinues = /* check for consecutive continues */;
        if (consecutiveContinues >= maxContinuesInARow) {
            return false;
        }
        
        return true;
    },
    
    handler: async (runtime, message, state, options, callback) => {
        // Generate continued response
        const response = await generateMessageResponse({
            runtime,
            context: /* special continuation context */,
            modelClass: ModelClass.LARGE
        });
        
        // Send continuation via callback
        await callback(response);
        return response;
    },
    
    examples: [/* examples */]
};
```

> **Important Note**: The CONTINUE action is specifically limited to a maximum of 3 consecutive uses. This prevents agents from getting stuck in infinite loops of self-continuation.

### IGNORE Action

Allows the agent to deliberately not respond to a message:

```typescript
const ignoreAction: Action = {
    name: "IGNORE",
    similes: ["STOP_TALKING", "STOP_CONVERSATION"],
    description: "Call this action if ignoring the user is the most appropriate response...",
    
    validate: async () => true,  // Always valid, decision made by the LLM
    
    handler: async () => true,   // No response needed
    
    examples: [
        [
            { user: "{{user1}}", content: { text: "Go screw yourself" } },
            { user: "{{user2}}", content: { text: "", action: "IGNORE" } }
        ]
    ]
};
```

### NONE Action

The default response action used for standard conversational replies:

```typescript
const noneAction: Action = {
    name: "NONE",
    similes: ["DEFAULT_RESPONSE", "NORMAL_REPLY"],
    description: "Standard conversational response with no special action",
    
    validate: async () => true,  // Always valid as the default fallback
    
    handler: async () => true,   // No special handling needed
    
    examples: [] // Not typically needed as this is the default
};
```

### Other Built-in Actions

- **FOLLOW_ROOM / UNFOLLOW_ROOM**: Control notification settings
- **MUTE_ROOM / UNMUTE_ROOM**: Control muting settings

## Advanced Action Techniques

### Action Chaining

Create sequences of actions that work together:

```typescript
const firstStepAction: Action = {
    name: "FIRST_STEP",
    // ...
    handler: async (runtime, message, state, options, callback) => {
        // Execute first step
        const firstStepResult = await performFirstStep();
        
        // Prepare data for second step
        const secondStepData = prepareSecondStepData(firstStepResult);
        
        // Create a follow-up message with the next action
        const followupMessage = {
            id: uuidv4(),
            userId: runtime.agentId,
            agentId: runtime.agentId,
            roomId: message.roomId,
            content: {
                text: "Proceeding to the next step...",
                action: "SECOND_STEP",
                actionData: secondStepData
            }
        };
        
        // Store the message
        await runtime.messageManager.createMemory(followupMessage);
        
        // Process the new action
        await runtime.processActions(message, [followupMessage], state, callback);
        
        return true;
    }
};
```

### Stateful Actions

Maintain state between action calls:

```typescript
const multistepAction: Action = {
    name: "MULTISTEP_ACTION",
    // ...
    handler: async (runtime, message, state, options, callback) => {
        // Cache key for this conversation
        const cacheKey = `multistep:${message.roomId}:${message.userId}`;
        
        // Get current step from cache
        let actionState = await runtime.cacheManager.get(cacheKey);
        if (!actionState) {
            // Initialize state
            actionState = {
                step: 1,
                data: {}
            };
        }
        
        // Handle current step
        switch (actionState.step) {
            case 1:
                // Handle step 1
                actionState.data.step1Result = await performStep1();
                actionState.step = 2;
                
                // Store updated state
                await runtime.cacheManager.set(cacheKey, actionState, 3600);
                
                // Prompt for next step
                await callback({
                    text: "Step 1 complete. Please provide information for step 2.",
                    action: null
                });
                break;
                
            case 2:
                // Handle step 2 using data from step 1
                const finalResult = await performStep2(actionState.data.step1Result, message);
                
                // Clear state when done
                await runtime.cacheManager.delete(cacheKey);
                
                // Send final result
                await callback({
                    text: `Process complete! Result: ${finalResult}`,
                    action: null
                });
                break;
        }
        
        return true;
    }
};
```

### Integration with External Services

Connect actions to external systems:

```typescript
const databaseAction: Action = {
    name: "QUERY_DATABASE",
    // ...
    handler: async (runtime, message, state, options, callback) => {
        // Extract query parameters
        const queryParams = extractQueryParams(message.content.text);
        
        // Get database service
        const dbService = runtime.getService(ServiceType.DATABASE);
        if (!dbService) {
            await callback({ 
                text: "Database service is not available", 
                action: null 
            });
            return false;
        }
        
        try {
            // Execute query
            const results = await dbService.executeQuery(queryParams);
            
            // Format results
            const formattedResults = formatQueryResults(results);
            
            // Send response
            await callback({
                text: `Here are the database results:\n\n${formattedResults}`,
                action: null
            });
            
            return true;
        } catch (error) {
            await callback({
                text: `Database query failed: ${error.message}`,
                action: null
            });
            return false;
        }
    }
};
```

### Custom Templates for Actions

Use specialized templates for complex actions:

```typescript
const analyticsAction: Action = {
    name: "ANALYZE_DATA",
    // ...
    handler: async (runtime, message, state, options, callback) => {
        // Extract data to analyze
        const dataToAnalyze = extractDataForAnalysis(message.content.text);
        
        // Specialized template for data analysis
        const analysisTemplate = `
# Data Analysis Template
You are performing specialized data analysis.

## Data to Analyze
{{data}}

## Analysis Instructions
1. Identify key trends in the data
2. Calculate relevant statistics
3. Summarize the most important findings
4. Provide actionable insights

Your analysis should be clear, concise, and focused on the most important aspects.
`;
        
        // Generate analysis using specialized template
        const analysisContext = composeContext({
            template: analysisTemplate,
            state: {
                data: JSON.stringify(dataToAnalyze, null, 2)
            }
        });
        
        const analysis = await generateText(runtime, analysisContext, {
            model: "gpt-4",
            temperature: 0.2,
            max_tokens: 1000
        });
        
        // Send response
        await callback({
            text: `## Data Analysis Results\n\n${analysis}`,
            action: null
        });
        
        return true;
    }
};
```

## Security and Validation

### Input Validation

Always validate user input before processing:

```typescript
validate: async (runtime: IAgentRuntime, message: Memory) => {
    // Check for required information
    const text = message.content.text.toLowerCase();
    
    // Ensure message contains required parameters
    if (!text.includes("transfer") || !text.includes("to")) {
        return false;
    }
    
    // Extract and validate amount
    const amountMatch = text.match(/(\d+(\.\d+)?)\s*(tokens|sol|eth)/i);
    if (!amountMatch) {
        return false;
    }
    
    // Extract and validate recipient
    const recipientMatch = text.match(/to\s+(@\w+|0x[a-fA-F0-9]{40}|[a-zA-Z0-9]{32,44})/i);
    if (!recipientMatch) {
        return false;
    }
    
    return true;
}
```

### Permission Checking

Verify the user has appropriate permissions:

```typescript
validate: async (runtime: IAgentRuntime, message: Memory) => {
    // Get user permissions
    const userPermissions = await runtime.databaseAdapter.getUserPermissions(message.userId);
    
    // Check for required permission
    if (!userPermissions.includes("ADMIN_ACCESS")) {
        return false;
    }
    
    return true;
}
```

### Rate Limiting

Prevent excessive action usage:

```typescript
validate: async (runtime: IAgentRuntime, message: Memory) => {
    // Cache key for rate limiting
    const rateLimitKey = `rate_limit:${this.name}:${message.userId}`;
    
    // Check current count
    const currentCount = parseInt(await runtime.cacheManager.get(rateLimitKey) || "0");
    
    // Apply rate limit
    if (currentCount >= 5) { // Limit to 5 per hour
        return false;
    }
    
    // Increment count
    await runtime.cacheManager.set(rateLimitKey, (currentCount + 1).toString(), 3600);
    
    return true;
}
```

### Error Handling

Implement robust error handling:

```typescript
handler: async (runtime: IAgentRuntime, message: Memory, state?: State, options?: any, callback?: HandlerCallback) => {
    try {
        // Action implementation
        const result = await performActionLogic();
        
        // Success response
        if (callback) {
            await callback({
                text: "Action completed successfully",
                action: null
            });
        }
        
        return true;
    } catch (error) {
        // Log the error
        console.error(`Action failed: ${error.message}`, error);
        
        // Provide user-friendly error response
        if (callback) {
            await callback({
                text: "I'm sorry, I couldn't complete that action. Please try again later.",
                action: null
            });
        }
        
        return false;
    }
}
```

## Best Practices

### Action Design

- **Single Responsibility**: Each action should do one thing well
- **Clear Purpose**: The action's purpose should be obvious from its name and description
- **Meaningful Examples**: Provide diverse examples showing exactly when to use the action
- **Descriptive Names**: Use clear, action-oriented names (e.g., `TRANSFER_TOKENS` not `TOKENS`)
- **Comprehensive Similes**: Include alternative phrases that might trigger the action

### Effective Similes

Similes play a crucial role in action detection. When designing similes:

1. **Use Verb-Noun Format**: Prefer `CHECK_WEATHER` over `WEATHER_INFO`
2. **Include Common Variations**: Add variations like `GET_WEATHER`, `SHOW_WEATHER`
3. **Consider Synonyms**: Include synonyms like `FORECAST` for weather-related actions
4. **Avoid Overlaps**: Ensure similes don't overlap with other actions
5. **Keep Them Relevant**: All similes should closely relate to the main action purpose

### Example Organization

Effective examples are organized to cover:

1. **Comprehensive Coverage**

```typescript
examples: [
    // Happy path - standard usage
    [basicUsageExample],
    // Edge cases - unusual but valid requests
    [edgeCaseExample],
    // Error cases - showing appropriate error handling
    [errorCaseExample],
];
```

2. **Clear Context**

```typescript
examples: [
    [
        {
            user: "{{user1}}",
            content: {
                text: "Context message showing why action is needed",
            },
        },
        {
            user: "{{user2}}",
            content: {
                text: "Clear response demonstrating action usage",
                action: "ACTION_NAME",
            },
        },
    ],
];
```

### Action Implementation

- **Robust Validation**: Thoroughly validate all inputs before processing
- **Consistent Error Handling**: Handle errors gracefully and provide helpful messages
- **Efficient Resource Usage**: Minimize database queries and API calls
- **Clear Responses**: Send clear, informative responses about action results
- **Idempotent Operations**: When possible, design actions to be safely repeatable

### Action Usage Guidelines

- **Context Appropriateness**: Only trigger actions when contextually appropriate
- **User Privacy**: Be careful with sensitive data
- **Progressive Enhancement**: Actions should enhance, not replace, conversational abilities
- **User Confirmation**: For impactful actions, consider confirming with the user first
- **Graceful Degradation**: Handle unavailable services or resources elegantly

## Testing Actions

### Unit Testing

Test action validation and handlers:

```typescript
describe("TransferTokensAction", () => {
    it("validates correctly", async () => {
        const runtime = mockRuntime();
        
        // Valid message
        const validMessage = { 
            content: { text: "Transfer 100 tokens to @user" } 
        };
        expect(await transferTokensAction.validate(runtime, validMessage)).toBe(true);
        
        // Invalid message - missing amount
        const invalidMessage1 = { 
            content: { text: "Transfer tokens to @user" } 
        };
        expect(await transferTokensAction.validate(runtime, invalidMessage1)).toBe(false);
        
        // Invalid message - missing recipient
        const invalidMessage2 = { 
            content: { text: "Transfer 100 tokens" } 
        };
        expect(await transferTokensAction.validate(runtime, invalidMessage2)).toBe(false);
    });
    
    it("handles transfers correctly", async () => {
        const runtime = mockRuntime();
        const message = { 
            content: { text: "Transfer 100 tokens to @user" } 
        };
        const callback = vi.fn();
        
        // Mock successful transfer
        walletService.transferTokens.mockResolvedValue("tx123");
        
        await transferTokensAction.handler(runtime, message, {}, {}, callback);
        
        // Check wallet service was called correctly
        expect(walletService.transferTokens).toHaveBeenCalledWith(100, "@user");
        
        // Check callback was called with success message
        expect(callback).toHaveBeenCalledWith(expect.objectContaining({
            text: expect.stringContaining("successfully transferred")
        }));
    });
    
    it("handles errors gracefully", async () => {
        const runtime = mockRuntime();
        const message = { 
            content: { text: "Transfer 100 tokens to @user" } 
        };
        const callback = vi.fn();
        
        // Mock failed transfer
        walletService.transferTokens.mockRejectedValue(new Error("Insufficient funds"));
        
        await transferTokensAction.handler(runtime, message, {}, {}, callback);
        
        // Check callback was called with error message
        expect(callback).toHaveBeenCalledWith(expect.objectContaining({
            text: expect.stringContaining("Insufficient funds")
        }));
    });
});
```

### Integration Testing

Test actions with the full runtime:

```typescript
describe("Action Integration", () => {
    let runtime: IAgentRuntime;
    
    beforeEach(async () => {
        // Set up full runtime with real dependencies
        runtime = await createTestRuntime();
        await runtime.registerAction(transferTokensAction);
    });
    
    it("processes action from response", async () => {
        // Mock response with action
        const userMessage = createMemory("Transfer 100 tokens to @user");
        const agentResponse = createMemory("I'll transfer those tokens for you.", "TRANSFER_TOKENS");
        
        // Process the action
        const result = await runtime.processActions(
            userMessage, 
            [agentResponse], 
            {} as State,
            async (newResponse) => {
                // Assertions about the callback response
                expect(newResponse.text).toContain("successfully transferred");
            }
        );
        
        // Assertions about the final result
        expect(result).toHaveLength(1);
        expect(result[0].content.text).toContain("successfully transferred");
    });
});
```

## Troubleshooting

### Common Issues

1. **Action Not Triggering**
   - Check validation logic - ensure it's returning true when appropriate
   - Verify similes list - add more variations of the action name
   - Review example patterns - ensure they cover the user's request pattern
   - Check action name matching - ensure normalized names match correctly

2. **Action Validation Failing**
   - Debug validation logic
   - Check if required parameters are present
   - Verify user permissions
   - Look for edge cases your validation might reject

3. **Action Handler Errors**
   - Implement detailed error logging
   - Add try/catch blocks around external API calls
   - Check for null/undefined values
   - Validate service availability before attempting to use it
   - Check state requirements are met
   - Review error logs for clues

4. **Response Not Sent**
   - Verify callback function is being called
   - Check response formatting
   - Ensure async operations complete before returning

5. **State Inconsistencies**
   - Verify state updates are saved properly
   - Check for concurrent modifications
   - Review state transitions for logical errors

### Debugging Techniques

```typescript
// Add debug information to your action
const debuggableAction: Action = {
    name: "DEBUG_ACTION",
    // ...
    validate: async (runtime: IAgentRuntime, message: Memory) => {
        console.log(`[DEBUG] Validating action ${this.name} for message: ${message.content.text}`);
        // Validation logic
        const result = /* validation logic */;
        console.log(`[DEBUG] Validation result: ${result}`);
        return result;
    },
    
    handler: async (runtime: IAgentRuntime, message: Memory, state?: State, options?: any, callback?: HandlerCallback) => {
        console.log(`[DEBUG] Executing action ${this.name}`);
        console.log(`[DEBUG] Message: ${JSON.stringify(message)}`);
        console.log(`[DEBUG] State: ${JSON.stringify(state)}`);
        console.log(`[DEBUG] Options: ${JSON.stringify(options)}`);
        
        try {
            // Action implementation
            const result = await performActionLogic();
            console.log(`[DEBUG] Action result: ${JSON.stringify(result)}`);
            
            if (callback) {
                const response = {
                    text: `Action completed with result: ${JSON.stringify(result)}`,
                    action: null
                };
                console.log(`[DEBUG] Sending response: ${JSON.stringify(response)}`);
                await callback(response);
            }
            
            return true;
        } catch (error) {
            console.error(`[DEBUG] Action error: ${error.message}`, error);
            
            if (callback) {
                const errorResponse = {
                    text: `Action failed: ${error.message}`,
                    action: null
                };
                console.log(`[DEBUG] Sending error response: ${JSON.stringify(errorResponse)}`);
                await callback(errorResponse);
            }
            
            return false;
        }
    }
};
```

---

By mastering action handling in the Eliza framework, you can create agents that go beyond simple conversation to perform complex tasks, integrate with external systems, and maintain sophisticated interaction patterns. Well-designed actions transform agents from passive responders into capable assistants that can truly help users accomplish their goals.

## Resources

- [Official Eliza Documentation](https://elizaos.github.io/eliza/docs/core/actions/)
- [Eliza GitHub Repository](https://github.com/elizaos/eliza/packages/core/src/types.ts)
- [ElizaOS Community](https://elizaos.github.io/eliza)
