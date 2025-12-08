# Pre-Select Tool Demo

A Spring Boot demonstration project showcasing an alternative approach to tool selection - pre-selecting tools based on conversation context BEFORE calling the LLM, rather than letting the LLM discover tools on-demand.

## Overview

This demo shows the `PreSelectToolCallAdvisor` which:
- Indexes all available tools using a `ToolSearcher`
- Searches for relevant tools based on the user's message text
- Pre-selects matching tools BEFORE sending to the LLM
- Only sends the pre-selected tools to the LLM (not all tools)

**Key Difference from Tool Search Tool:**
- `ToolSearchToolCallAdvisor`: LLM actively discovers tools by calling `toolSearchTool`
- `PreSelectToolCallAdvisor`: Tools are pre-selected based on message content before LLM sees them

## Prerequisites

- Java 17 or higher
- Maven 3.6+
- API key for your preferred LLM (configured in `application.properties`)

## Running the Demo

```bash
cd examples/pre-select-tool-demo
mvn spring-boot:run
```

## Key Components

### 1. PreSelectToolCallAdvisor

Extends `ToolCallAdvisor` to pre-select tools based on conversation context:

```java
@Override
protected ChatClientRequest doBeforeCall(ChatClientRequest request, CallAdvisorChain chain) {
    // Extract user message text
    String userMessagesText = request.prompt().getUserMessages()
        .stream()
        .map(m -> m.getText())
        .collect(Collectors.joining("\n"));

    // Search for relevant tools based on message content
    ToolSearchResponse response = toolSearcher.search(
        new ToolSearchRequest(sessionId, userMessagesText, null, null));

    // Pre-select only the matching tools
    toolOptionsCopy.setToolCallbacks(selectedToolCallbacks);
    
    return augmentedRequest;
}
```

### 2. Main Application

Demonstrates a two-step "Chain of Thought" approach:

```java
ChatClient chatClient = chatClientBuilder
    .defaultTools(new MyTools())
    .defaultAdvisors(PreSelectToolCallAdvisor.builder()
        .toolSearcher(toolSearcher)
        .build())
    .defaultAdvisors(ToolCallAdvisor.builder().build())
    .build();

// Step 1: Ask for reasoning steps and required tools
var answer1 = chatClient.prompt("""
    Please provide short summary of the reasoning steps and required tools 
    you would use to answer the following question:
    ---
    What should I wear right now in Landsmeer, NL.
    Please suggest some budget-friendly clothing shops that are open.
    ---
    """).call().content();

// Step 2: Use the CoT response (which mentions tool names) as the prompt
var answer = chatClient.prompt(answer1).call().content();
```

## The Challenge: Why Chain of Thought?

**Problem:** Direct user messages often don't contain keywords that match tool names.

| User Message | Tool Needed | Match? |
|--------------|-------------|--------|
| "What should I wear?" | `weather`, `clothing` | ❌ No match |
| "Get me the weather" | `weather` | ✅ Matches |

**Solution:** Use a preliminary LLM call to generate a "plan" that explicitly mentions the tools needed.

```
User: "What should I wear in Amsterdam?"
         ↓
LLM (CoT): "To answer this, I need to:
           1. Get current time using currentTime tool
           2. Check weather using weather tool
           3. Find clothing shops using clothing tool"
         ↓
Pre-Select: Matches "currentTime", "weather", "clothing" → selects those tools
         ↓
LLM (with selected tools): Executes the plan
```

## Expected Flow

1. **First Request** - Ask for reasoning/plan
   - User: "Please provide reasoning steps and required tools..."
   - LLM: "I'll need to use: currentTime, weather, clothing tools..."

2. **Second Request** - Execute with pre-selected tools
   - Input: The LLM's plan (contains tool names)
   - Pre-select: Searches and finds `currentTime`, `weather`, `clothing`
   - LLM receives only those 3 tools (not all tools)
   - LLM executes the tools and generates answer

## Limitations

| Aspect | Pre-Select Approach | Tool Search Tool Approach |
|--------|---------------------|---------------------------|
| Works with direct user questions | ❌ Limited | ✅ Yes |
| Requires preliminary LLM call | ✅ Yes (for best results) | ❌ No |
| LLM actively discovers tools | ❌ No | ✅ Yes |
| Token efficiency | ⚠️ Medium (2 LLM calls) | ✅ High |
| Search accuracy | ⚠️ Depends on message content | ✅ LLM-guided |

## When to Use

**Good fit for Pre-Select:**
- Messages naturally contain tool-related keywords
- You want deterministic tool selection (no LLM involvement in discovery)
- Building pipelines where you control the input format

**Better to use Tool Search Tool when:**
- User messages are natural language without tool keywords
- You want the LLM to intelligently discover needed tools
- Building conversational interfaces

## Sample Tools

```java
@Tool(description = "Get the current weather for a given location")
public String weather(String location, String atTime) { ... }

@Tool(description = "Get clothing shops names for a given location")
public List<String> clothing(String location, String openAtTime) { ... }

@Tool(description = "Provides the current date and time for a given location")
public String currentTime(String location) { ... }
```

## Related Documentation

- [Tool Search Tool README](../../tool-search-tool/README.md)
- [Tool Searchers README](../../tool-searchers/README.md)
- [Tool Search Tool Demo](../tool-search-tool-demo/README.md) - The recommended approach
