---
title: "Cypher: Don't just generate, Iterate!"
date: "2025/08/25 00:00:00"
---

Current LLMs are pretty good at generating cypher. They do get it wrong sometimes and we need to nudge it back to the correct path. Continuing from my previous article about [Cypher validation](https://www.linkedin.com/pulse/calling-python-from-java-ai-framework-aldrin-misquitta-2ro5f), here is one way to use this feedback to generate better cypher. 

In essence, we wish to achieve the following workflow in Embabel,

{% asset_img cypher-correction-workflow.png 'Cypher generation workflow' %}

This can be achieved with pre and post conditions. Intuitively, `pre` indicates conditions that must be met before that particular action is invoked and `post` indicates an update to the condition after the action is complete.

*(Complete code mentioned here can be found at https://github.com/aldrinm/cypher-error-correction)*

Here is the action for an agent that begins the cypher generation,

```
@Action(
  description = "Generates a cypher statement that may be used to answer the user's query ",
  post = VALIDATE_CYPHER_NEEDED
)
CypherStatementRequest generateCypher(UserInput userInput)
```
This action indicates that the condition VALIDATE_CYPHER_NEEDED needs to be evaluated. The condition is described as,

```
@Condition(name = *VALIDATE_CYPHER_NEEDED*)
public Boolean validateCypherNeeded(OperationContext context) {
  var last = context.lastResult();
  return last instanceof CypherStatementRequest;
}
```

Similarly, other conditions are defined by methods annotated with `@Condition`.

### The key part
The `validateCypher` action validates the cypher by calling local tools

```
@Action(
  description = "Validates the given cypher and generates a report",
  post = {CYPHER_VALID, CYPHER_NOT_VALID},
  canRerun = true,
  pre = VALIDATE_CYPHER_NEEDED
)
public ValidationReport validateCypher(CypherStatementRequest cypherStatementRequest)
```

This action is marked as `canRerun = true` - allowing it to be called as many times as needed.

If the cypher is invalid, it causes the `rectifyCypher` action to jump in. This action uses the feedback in the `ValidationReport`, sends it to the LLM and the LLM will attempts to correct the cypher,
 
```
@Action(
  pre = CYPHER_NOT_VALID,
  description = "Tries to correct the cypher based on feedback",
  post = { VALIDATE_CYPHER_NEEDED },
  canRerun = true
)
public CypherStatementRequest rectifyCypher(CypherStatementRequest cypherStatementRequest, ValidationReport validationReport) 
```

Notice this too is marked `canRerun = true`. `validateCypher` and `rectifyCypher` can be invoked as many times as needed in order to achieve the end goal. 
This is the key part - `rectifyCypher` and `validateCypher` are sequentially run in a loop till a valid cypher is produced (or till the global maximum actions is exceeded)


Finally, if all is good, the execute cypher action is invoked,
```
@Action(
  description = "Executes a cypher statement ",
  pre = CYPHER_VALID
)
CypherExecutionResult executeCypher(CypherStatementRequest cypherStatementRequest)
      
```

The `executeCypher` has a pre-condition `CYPHER_VALID` so that control lands here only if the cypher is valid.

 Of course it may be possible that the agent may get trapped into an infinite loop if it cannot generate correct cypher. The agent's action can be configured to limit the max attempts (by default I think it is 50). See `ProcessOptions` https://github.com/embabel/embabel-agent/blob/main/embabel-agent-api/src/main/kotlin/com/embabel/agent/core/ProcessOptions.kt

Here for example is a test run where the cypher was fixed in one attempt, by correcting the property name and the relationship type:

{% asset_img test-run.png 'A test run' %}


Finally, I had some trouble with Neo4j's [MCP Server](https://github.com/neo4j-contrib/mcp-neo4j/tree/main/servers/mcp-neo4j-cypher). Running a read only query would often fail with an error `Error executing tool read_neo4j_cypher: Only MATCH queries are allowed for read-query`. Even though it was clearly a read-only query. Turns out I had a node label `:Set` that was triggering the read-only check. This is a [pre-existing issue](https://github.com/neo4j-contrib/mcp-neo4j/issues/56https://github.com/neo4j-contrib/mcp-neo4j/issues/56) in the tool. I got around this one by injecting the `Neo4jClient` and executing the query directly instead.
