---
source: aidevs4-lesson3
---
## 1. The Core Paradigm Shift: Workflow vs. Agent

Implementation requires choosing between two logic-control methods:

- **Workflow:** Logic is hardcoded by the programmer (if/else, loops). The LLM is just a "worker" at specific steps. This is best for deterministic, high-stakes processes like accounting.
    
- **Agent:** Logic is controlled by the LLM at runtime. The model decides which tool to use next based on the result of the previous one. This is better for complex, unpredictable tasks.
    
- **Implementation Advice:** Don't use an Agent where a simple script (Workflow) suffices. Agents are "probabilistic" and more expensive/slower.
    

## 2. Model Context Protocol (MCP) Architecture

MCP is an open standard that decouples the AI application from the data sources.

- **Host:** The main application (e.g., Claude Desktop, your Python app) where the LLM "lives."
    
- **Client:** A component within the Host that speaks the MCP protocol.
    
- **Server:** A lightweight program that exposes specific **Tools**, **Resources**, and **Prompts**.
    
- **Implementation Advice:** Use **STDIO** for local integrations (the Host starts the Server as a child process) and **SSE (Server-Sent Events) over HTTP** for remote/cloud-based tools.
    

## 3. Tools vs. Resources

- **Resources (Passive):** Data that the model can read (like a file or a documentation page). They are identified by URIs (e.g., `postgres://db/schema`).
    
- **Tools (Active):** Executable functions that can change the state of the world (e.g., `send_email`, `write_file`).
    
- **Implementation Advice:** Do not turn everything into a Tool. If the model just needs to "know" something, expose it as a Resource to save the model from having to "decide" to call a function.
    

## 4. Designing Tool Granularity

A common mistake is exposing too many "atomic" tools (e.g., `create_file`, `append_file`, `move_file`).

- **The Problem:** Too many tools overwhelm the model's context window and increase the "cumulative probability of failure" (the more steps, the higher the chance of a hallucination).
    
- **The Solution:** Aggregate tools into logical blocks (e.g., a single `file_manager` tool that handles multiple operations).
    
- **Implementation Advice:** Aim for high-level tools that perform a complete "logical unit of work" rather than low-level API wrappers.
    

## 5. Implementation Best Practices for Stability

### Recovery Hints (Error Handling)

Standard API errors (e.g., `400 Bad Request`) are useless to an LLM.

- **Advise:** When a tool fails, return a "Hint." Instead of "Invalid ID," return: _"Error: ID '123' not found. Available IDs are: [List]. Please try again with a valid ID."_ This allows the agent to self-correct without human intervention.
    

### Optimistic Locking (Checksums)

In agentic workflows, a file might change between the time the model reads it and the time it writes it.

- **Advise:** Implement **Checksums**. Have your `read` tool return a hash of the file. Require that hash to be sent back with the `write` tool. If the hashes don't match, the server should reject the change and tell the model to "re-read" the latest version.
    

### Dry Runs

- **Advise:** For "destructive" tools (delete, overwrite), implement a `dryRun: boolean` parameter. The model can see the intended result before committing, increasing safety.
    

## 6. Advanced MCP Features

### Sampling (Model-invoked Model)

This allows the **Server** to ask the **Host** to run an LLM completion.

- **Advise:** Use Sampling when a tool needs "intelligence" to complete its task (e.g., a "code_reviewer" tool that needs an LLM to explain why a piece of code is buggy). This keeps the complex logic inside the server while using the Host's API keys and security settings.
    

### Elicitation

If a model calls a tool with missing or ambiguous parameters:

- **Advise:** The tool should return a request for clarification rather than failing. This keeps the "Agentic loop" alive.
    

### Pagination

- **Advise:** Never return massive datasets. If a tool queries a database, return the first 20 rows and a `cursor` or `next_page_id`. This prevents the model's context from being flooded with irrelevant data.
