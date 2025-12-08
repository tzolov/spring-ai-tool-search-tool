# Tool Search Tool Demo

A Spring Boot demonstration project showcasing the **Tool Search Tool** pattern for Spring AI - enabling dynamic, on-demand tool discovery instead of loading all tool definitions upfront.

## Overview

This demo shows how the `ToolSearchToolCallAdvisor` allows an LLM to discover and use tools on-demand, significantly reducing token usage when working with large tool libraries.

## Prerequisites

- Java 17 or higher
- Maven 3.6+
- API key for your preferred LLM (configured in `application.properties`)

## Running the Demo

```bash
cd examples/tool-search-tool-demo
mvn spring-boot:run
```

## Key Components

### 1. Main Application (`Application.java`)

Demonstrates the Tool Search Tool pattern:

```java
var toolSearchToolCallAdvisor = ToolSearchToolCallAdvisor.builder()
    .toolSearcher(toolSearcher)
    .build();

ChatClient chatClient = chatClientBuilder
    .defaultTools(new MyTools())  // Tools registered but NOT sent to LLM initially
    .defaultAdvisors(toolSearchToolCallAdvisor)
    .defaultAdvisors(new MyLoggingAdvisor())
    .build();

var answer = chatClient.prompt("""
    Help me plan what to wear today in Landsmeer, NL.
    Please suggest clothing shops that are open right now in the area.
    """).call().content();
```

**Key aspects:**
- `ToolSearchToolCallAdvisor`: Intercepts tool calling to enable on-demand discovery
- Tools are indexed but NOT sent to the LLM initially - only the search tool is provided
- LLM discovers tools by calling `toolSearchTool` as needed

### 2. Sample Tools (`MyTools` class)

```java
@Tool(description = "Get the current weather for a given location and at a given time")
public String weather(String location, String atTime) {
    return "The current weather in " + location + " is sunny with a temperature of 25°C.";
}

@Tool(description = "Get of clothing shops names for a given location")
public List<String> clothing(String location, String openAtTime) {
    return List.of("Foo", "Bar", "Baz");
}

@Tool(description = "Provides the current date and time for a given location")
public String currentTime(String location) {
    return LocalDateTime.now().toString();
}
```

These tools are discovered on-demand by the LLM, not loaded upfront.

### 3. Configuration (`Config.java`)

Configures the `ToolSearcher` implementation. Options include:
- `VectorToolSearcher` - Semantic search using embeddings
- `LuceneToolSearcher` - Keyword-based full-text search
- `RegexToolSearcher` - Pattern matching

### 4. Logging Advisor (`MyLoggingAdvisor`)

Logs each iteration to show the progressive tool discovery:

```java
@Override
public ChatClientRequest before(ChatClientRequest request, AdvisorChain chain) {
    print("REQUEST", request.prompt().getInstructions());
    return request;
}
```

## Expected Flow

When running the demo, you'll see the tool discovery process:

1. **First Request** - LLM receives only `toolSearchTool`
   - LLM calls `toolSearchTool(query="current time")` → discovers `currentTime`

2. **Second Request** - LLM sees `toolSearchTool` + `currentTime`
   - LLM calls `currentTime("Landsmeer, NL")` → returns current time
   - LLM calls `toolSearchTool(query="weather")` → discovers `weather`

3. **Third Request** - LLM sees `toolSearchTool` + `currentTime` + `weather`
   - LLM calls `weather("Landsmeer, NL", "...")` → returns weather
   - LLM calls `toolSearchTool(query="clothing shops")` → discovers `clothing`

4. **Fourth Request** - LLM sees all discovered tools
   - LLM calls `clothing("Landsmeer, NL", "...")` → returns shop list

5. **Final Response** - LLM generates clothing recommendations

## Sample Output

```
REQUEST: TOOLS: ["toolSearchTool"]
text: "Help me plan what to wear today in Landsmeer, NL..."

RESPONSE:
- toolSearchTool({"query":"current time and date"})

REQUEST: TOOLS: ["toolSearchTool","currentTime","weather"]
- toolSearchTool -> ["currentTime","weather"]

RESPONSE:
- currentTime({"location":"Landsmeer, NL"})

REQUEST: TOOLS: ["toolSearchTool","currentTime","weather"]
- currentTime -> "2025-12-08T12:30:26"

RESPONSE:
- weather({"location":"Landsmeer, NL","atTime":"2025-12-08T12:30"})
- toolSearchTool({"query":"clothing shops"})

...

FINAL:
"Based on the sunny 25°C weather in Landsmeer, NL, I recommend light layers.
Here are clothing shops open now: Foo, Bar, Baz..."
```

## Related Documentation

- [Tool Search Tool README](../../tool-search-tool/README.md)
- [Tool Searchers README](../../tool-searchers/README.md)
- [Spring AI Recursive Advisors](https://docs.spring.io/spring-ai/reference/api/advisors-recursive.html)
