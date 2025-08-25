---
title: Calling Python from a Java AI framework
date: "2025/08/15 00:00:00"
---


One convenient thing that you can do with MCP(Model context protocol) tools is use libraries from a different programming language. 

I've been trying the [Embabel](https://github.com/embabel/embabel-agent) framework, the new AI agent framework created by Rod Johnson. I set up a Neo4j instance and created an agent to build cypher statements from natural language queries. Executing the cypher and presenting the results would complete the goal of the agent. This is one way to do RAG (Retrieval-augmented generation) with a graph.

If you feed it the database schema, any decent LLM will generate a reasonably okay cypher statement. However, LLMs often generate cypher that fail because of improper syntax or return no results because of wrong choice of node labels or relationships or wrong properties in the clauses. 

I wanted to add a verification step before executing the cypher. There is a neat library [CyVer] (https://gitlab.com/netmode/CyVer) that can perform syntax, schema and property checks on cypher. It is written in Python. So to be able to include it in my agent I wrapped it into an MCP tool. Here's the script for the MCP server https://github.com/aldrinm/cyver-mcp

Exposing the tool to the LLM and allowing tool callback worked very well and I was able to get feedback on the generated cypher statements. But I didn't like the roundabout way of invoking a local tool through an LLM. There must be a way to invoke it explicitly and I was able to find this through the `McpSyncClient`.


Like so, (relevant code only),

{% codeblock line_number:false%}

private final List<McpSyncClient> mcpSyncClients; //Inject
...
//Iterate and find the mcp server
Optional<McpSyncClient> cyver = mcpSyncClients.stream().filter(c -> c.getServerInfo().name().equals("cyver")).findAny();

ToolCallback callback = new SyncMcpToolCallback(cyverClient, tool.get());
//JSON argument is the parameter that the function requires. If no parameters, just us "{}"
String result = callback.call("{\"query\": \"" + cypherStatement + "\"}"); 

//Easier to handle the results if you map it to a class,
final ObjectMapper objectMapper = new ObjectMapper();
objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
try {
  List<Map<String, Object>> listResult = objectMapper.readValue(result, List.class);
  // Assuming the first element of the list contains the actual data
  if (!listResult.isEmpty()) {
    Map<String, Object> dataMap = listResult.get(0);
    String textValue = (String) dataMap.get("text");
    return objectMapper.readValue(textValue, CyverSyntaxValidatonResult.class);
  }
} catch (JsonProcessingException ex) {
  throw new RuntimeException("Failed to parse result", ex);
}
{% endcodeblock %}

​
This worked rather well. I was also to call [Neo4j's MCP Server](https://github.com/neo4j-contrib/mcp-neo4j/tree/main/servers/mcp-neo4j-cypher) to obtain the schema to feed it into the first prompt. This should also work in [Spring AI](https://spring.io/projects/spring-ai) projects, considering that Embabel is built on top of Spring AI.
On the whole, this saved me some token usage and gave me a whole lot of code and a maintenance hassle. Though a very satisfying academic exercise!


*All code for this article is at https://github.com/aldrinm/embabel-neo4j-cyver-mcp*

What are some of the hacks you've used with Graph RAG, cypher or Neo4j and LLMs? Would love to hear about it.