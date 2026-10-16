# Liferay AI Hub Guide

## Core Concepts

Liferay AI Hub is a low-code SaaS environment that brings agentic orchestration to Liferay DXP. It leverages Large Language Models (LLMs) and Model Context Protocol (MCP) servers to automate workflows and integrate AI into the content lifecycle.

### Key Features
1. **Built-in AI Writing Tasks**: Tone adjustment, grammar correction, and summarization.
2. **Content Generation**: Capabilities for content translation, image generation, and automated asset categorization.
3. **Custom Agents**: Users can build custom agents with specific data sources and guardrails.
4. **AI Assistant Chatbot**: A centralized chatbot that coordinates complex tasks and searches the DXP instance for contextualized answers.
5. **Administration**: Tools for managing environments, API credentials, token usage, and activity monitoring.

## Architecture

Liferay AI Hub integrates with DXP instances by connecting to external Large Language Models and utilizing MCP (Model Context Protocol) servers to safely fetch local context or perform actions on behalf of the user.

### Chatbot Client Extensions

AI Hub Chatbots can be backed by Server-Side Client Extensions (like a Spring Boot microservice or Node.js app). 
- **Security Paradigm**: Instead of exposing Admin APIs directly to the frontend, a custom backend acts as the "brain" for the chatbot.
- **Workflow**:
    1. A standard user interacts with the AI Hub Chatbot on the frontend.
    2. The frontend passes the message securely to the custom Client Extension (including a JWT token identifying the user).
    3. The Client Extension uses its own admin-level OAuth2 token (Client Credentials flow) to call Liferay Admin APIs securely, perform logic (like fetching contextual pricing or account details), and return the formatted answer to the user.

---

## Agent Deep Dive: Kaleo XML Under the Hood

Liferay AI Hub Agents are not built on a proprietary engine from scratch; they are a clever extension of Liferay's battle-tested **Kaleo Workflow Engine**. Every agent is fundamentally a `<workflow-definition>` XML file.

### 1. The Core AI Nodes
Instead of standard human approval tasks, AI Hub agent workflows string together specialized XML nodes:
*   **`<llm>` Node**: The primary execution engine. It defines standard `input-variables` and `output-variables`, formats a strict `<prompt>`, merges the user input via `<user-message>`, executes against the configured LLM, and pushes the answer into the designated output variable.
*   **API / Task Nodes**: Agents can make live `GET`/`POST` calls back into the host Liferay instance (often utilizing an injected `aiHubCellLiferayDXPURL` variable). This allows an agent to fetch headless JSON data, feed it into an `<llm>` node for analysis, and even post a result back (e.g., automatically applying a category to an asset).

### 2. The Chatbot Request/Response Lifecycle
When an AI Hub Chatbot is mapped to an Agent (like the `search_agent`), the interaction follows a strict variable contract:
1. The chatbot injects the user's text into an input variable (typically named **`request`**).
2. The agent's workflow starts, and the `<llm>` node consumes the `request` variable within its `<user-message>` template.
3. The LLM executes, and the result is stored in an output variable (typically named **`response`**).
4. The workflow terminates, and the chatbot reads the `response` variable to display the final answer in the UI.

### 3. RAG (Retrieval-Augmented Generation) Implementation
Agents can ground their answers in actual Liferay data using the `<rag>` block within an `<llm>` node. 

```xml
<rag>
    <![CDATA[
    {
        "contentRetriever": {
            "key": "liferay"
        }
    }
    ]]>
</rag>
```
If a valid RAG configuration is present, AI Hub intercepts the workflow execution *before* the prompt hits the LLM. It performs a semantic search against Liferay's Elasticsearch index using the user's input, retrieves relevant Web Content, Knowledge Base articles, or Documents, and secretly injects this context into the LLM prompt. If the RAG block is empty (`[]`), the LLM relies solely on its baseline training and explicitly provided input variables.

### 4. Guardrails and Strict Data Typing
Agents designed for headless automation (like Auto-Categorization or Content Gap Analysis) use intensive prompt engineering (Guardrails) to force the LLM to behave like a deterministic software function.
*   The prompt often demands a strict JSON output shape (e.g., *"Output a SINGLE raw JSON object and nothing else. DO NOT wrap the JSON in markdown fences"*).
*   This ensures the resulting output variable can be reliably parsed by a subsequent API node or external service without breaking JSON syntax.

---

## Complete XML Reference Template

To generate an AI Hub Agent from scratch, you MUST adhere to the following Kaleo Workflow XML schema. Pay strict attention to the `<llm>` node structure, CDATA wrapping for JSON arrays, and the transition routing.

```xml
<?xml version="1.0"?>
<workflow-definition
        xmlns="urn:liferay.com:liferay-workflow_7.4.0"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="urn:liferay.com:liferay-workflow_7.4.0 http://www.liferay.com/dtd/liferay-workflow-definition_7_4_0.xsd"
>
        <name>Agent Name</name>
        <version>1</version>
        
        <!-- LLM Node -->
        <llm>
                <name>myLlmNode</name>
                <description>Description of the LLM node task.</description>
                <labels>
                        <label language-id="en_US">My LLM Task</label>
                </labels>
                
                <!-- Define inputs (injected by chatbot or previous nodes) -->
                <input-variables>
                        <![CDATA[
[
        {
                "name": "request",
                "type": "string"
        }
]
]]>
                </input-variables>
                
                <!-- Define outputs -->
                <output-variables>
                        <![CDATA[
[
        {
                "name": "response",
                "type": "string"
        }
]
]]>
                </output-variables>
                
                <!-- System Prompt / Instructions -->
                <prompt>
                        <![CDATA[You are an AI assistant. You answer questions directly and concisely.]]>
                </prompt>
                
                <!-- RAG Config (Empty array [] if no RAG) -->
                <rag>
                        <![CDATA[
{
        "contentRetriever": {
                "key": "liferay"
        }
}
]]>
                </rag>
                
                <tools><![CDATA[[]]]></tools>
                
                <!-- Transition to next node (or end) -->
                <transitions>
                        <transition>
                                <labels>
                                        <label language-id="en_US">End</label>
                                </labels>
                                <name>end</name>
                                <target>end</target>
                                <default>true</default>
                        </transition>
                </transitions>
                
                <!-- User Message Template (injects variables via {{name}}) -->
                <user-message>
                        <![CDATA[{{request}}]]>
                </user-message>
        </llm>
        
        <!-- Mandatory Start State -->
        <state>
                <name>start</name>
                <initial>true</initial>
                <labels>
                        <label language-id="en_US">Start</label>
                </labels>
                <transitions>
                        <transition>
                                <labels>
                                        <label language-id="en_US">Start LLM</label>
                                </labels>
                                <name>myLlmNode</name>
                                <target>myLlmNode</target>
                                <default>true</default>
                        </transition>
                </transitions>
        </state>
        
        <!-- Mandatory End State -->
        <state>
                <name>end</name>
                <metadata>
                        <![CDATA[{"terminal": true}]]>
                </metadata>
                <labels>
                        <label language-id="en_US">End</label>
                </labels>
        </state>
</workflow-definition>
```

---

## Developer Documentation Extracts

*   **Agent Workflows:** Built in the Liferay Workflow Designer. They add `LLM`, `AI Decision`, and `API Execution` nodes to the standard palette.
*   **AI Decision Nodes:** Sends a prompt to an LLM and uses the answer to determine which path the workflow takes next.
*   **Data Sources:** Attach external URLs to agents. AI Hub crawls the linked website, stores extracted text, and makes it available to agents via semantic search for RAG.
*   **Chatbot Deployment:** Chatbots can be deployed via a raw JavaScript snippet (using the `External Reference Code`) or natively through DXP's "Click to Chat" configuration panel. Chatbots act as supervisors handling multi-turn conversations across their assigned agents.