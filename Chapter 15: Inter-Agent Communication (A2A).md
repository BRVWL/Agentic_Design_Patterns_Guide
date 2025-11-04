# Chapter 15: Inter-Agent Communication (A2A)

In the evolving field of artificial intelligence, individual agents, regardless of their capabilities, may encounter limitations when addressing intricate, multifaceted challenges. **Inter-Agent Communication (A2A)** facilitates the collaboration of distinct AI agents, potentially constructed with diverse frameworks, allowing them to seamlessly coordinate, delegate tasks, and exchange information. Google's A2A protocol is an open standard designed to enable this collaboration through a universal HTTP-based protocol. This chapter examines A2A, its practical applications, and its implementation within the Google Agent Development Kit (ADK).

## Inter-Agent Communication Pattern Overview

The Agent2Agent (A2A) protocol is an open standard that facilitates communication and collaboration between diverse AI agent frameworks. A2A enables interoperability among AI agents built with technologies such as LangGraph, CrewAI, or Google ADK, allowing them to work together regardless of their development origins or framework differences. This protocol is supported by various technology companies and service providers, including Atlassian, Box, LangChain, MongoDB, Salesforce, SAP, and ServiceNow. Microsoft has committed to supporting open protocols like A2A, with planned integration into Azure AI Foundry and Copilot Studio. Auth0 and SAP are integrating A2A support into their platforms and agents. The protocol is open-source and accepts community contributions to facilitate its evolution and widespread adoption.

### Core Concepts of A2A

The A2A protocol establishes a framework for agent interactions through defined core concepts. Understanding these concepts is essential for developing or integrating with A2A-compliant systems.

- **Core Actors:** A2A involves three main entities:
  - **User:** Initiates requests for agent assistance.
  - **A2A Client (Client Agent):** An application or AI agent that acts on the user's behalf to request actions or information.
  - **A2A Server (Remote Agent):** An AI agent or system that provides an HTTP endpoint to process client requests and return results. The remote agent operates as an "opaque" system, meaning the client does not need to understand its internal operational details.
- **Agent Card:** The Agent Card functions as an agent's digital identity. It is typically a JSON file that declares the agent's identity, endpoint URL, version, supported capabilities such as streaming or push notifications, specific skills, default input/output modes, and authentication requirements. Clients use Agent Cards for automatic discovery and to securely interact with agents. An example of an Agent Card for a WeatherBot is provided in Fig.1.

```json
{
  "name": "WeatherBot",
  "description": "Provides accurate weather forecasts and historical data.",
  "url": "http://weather-service.example.com/a2a",
  "version": "1.0.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": false,
    "stateTransitionHistory": true
  },
  "authentication": {
    "schemes": ["apiKey"]
  },
  "defaultInputModes": ["text"],
  "defaultOutputModes": ["text"],
  "skills": [
    {
      "id": "get_current_weather",
      "name": "Get Current Weather",
      "description": "Retrieve real-time weather for any location.",
      "inputModes": ["text"],
      "outputModes": ["text"],
      "examples": [
        "What's the weather in Paris?",
        "Current conditions in Tokyo"
      ],
      "tags": ["weather", "current", "real-time"]
    },
    {
      "id": "get_forecast",
      "name": "Get Forecast",
      "description": "Get 5-day weather predictions.",
      "inputModes": ["text"],
      "outputModes": ["text"],
      "examples": [
        "5-day forecast for New York",
        "Will it rain in London this weekend?"
      ],
      "tags": ["weather", "forecast", "prediction"]
    }
  ]
}
```

_Fig. 1.: Agent Card for a WeatherBot_

- **Agent Discovery:** it enables clients to locate Agent Cards, which detail the capabilities of available A2A Servers. Strategies for agent discovery include the **Well-Known URI** approach, wherein agents host their Agent Card at a standardized path (e.g., `/.well-known/agent.json`) for broad, often automated, public or domain-specific accessibility. **Curated Registries** provide a centralized catalog where Agent Cards are published and can be queried based on specific criteria, making them suitable for enterprise environments that require centralized management and access control. **Direct Configuration** involves embedding or privately sharing Agent Card information, appropriate for closely coupled or private systems where dynamic discovery is not essential. Regardless of the method chosen, securing Agent Card endpoints using access control, mutual TLS (mTLS), or network restrictions is important, particularly if the card contains sensitive, though non-secret, information.
- **Communications and Tasks:** In the Agent-to-Agent (A2A) framework, communication is structured around asynchronous **tasks**, which represent the fundamental units of work for long-running processes. Each task is assigned a unique identifier and moves through a series of states—such as `submitted`, `working`, or `completed`—a design that supports parallel processing in complex operations. Communication between agents occurs through a **Message**. This complete communication package contains **Attributes**, which are key-value metadata describing the message (like its priority or creation time), and one or more **Parts**, which carry the actual content being delivered, such as plain text, files, or structured JSON data. The tangible outputs generated by an agent during a task are called **Artifacts**. Like messages, artifacts are also composed of one or more parts and can be streamed incrementally as results become available. All communication within the A2A framework is conducted over HTTP(S) using the JSON-RPC 2.0 protocol for payloads. To maintain continuity across multiple interactions, a server-generated `contextId` is used to group related tasks and preserve context.
- **Interaction Mechanisms:** Request/Response (Polling) Server-Sent Events (SSE). A2A provides multiple interaction methods to suit a variety of AI application needs, each with a distinct mechanism:
  - **Synchronous Request/Response:** For quick, immediate operations. In this model, the client sends a request and actively waits for the server to process it and return a complete response in a single, synchronous exchange.
  - **Asynchronous Polling:** Suited for tasks that take longer to process. The client sends a request, and the server immediately acknowledges it with a "working" status and a task ID. The client is then free to perform other actions and can periodically poll the server by sending new requests to check the status of the task until it is marked as "completed" or "failed."
  - **Streaming Updates (Server-Sent Events - SSE):** Ideal for receiving real-time, incremental results. This method establishes a persistent, one-way connection from the server to the client. It allows the remote agent to continuously push updates, such as status changes or partial results, without the client needing to make multiple requests.
  - **Push Notifications (Webhooks):** Designed for very long-running or resource-intensive tasks where maintaining a constant connection or frequent polling is inefficient. The client can register a webhook URL, and the server will send an asynchronous notification (a "push") to that URL when the task's status changes significantly (e.g., upon completion).

The Agent Card specifies whether an agent supports streaming or push notification capabilities. Furthermore, A2A is modality-agnostic, meaning it can facilitate these interaction patterns not just for text, but also for other data types like audio and video, enabling rich, multimodal AI applications. Both streaming and push notification capabilities are specified within the Agent Card. A2A is modality agnostic, meaning it supports interactions beyond text, including audio and video streaming, facilitating multimodal AI applications.

```json
#Synchronous Request Example
{
    "jsonrpc": "2.0",
    "id": "1",
    "method": "sendTask",
    "params": {
        "id": "task-001",
        "sessionId": "session-001",
        "message": {
            "role": "user",
            "parts": [
                {
                    "type": "text",
                    "text": "What is the exchange rate from USD to EUR?"
                }
            ]
        },
        "acceptedOutputModes": ["text/plain"],
        "historyLength": 5
    }
}
```

The synchronous request uses the `sendTask` method, where the client asks for and expects a single, complete answer to its query. In contrast, the streaming request uses the `sendTaskSubscribe` method to establish a persistent connection, allowing the agent to send back multiple, incremental updates or partial results over time.

```json
# Streaming Request Example
{
    "jsonrpc": "2.0",
    "id": "2",
    "method": "sendTaskSubscribe",
    "params": {
        "id": "task-002",
        "sessionId": "session-001",
        "message": {
            "role": "user",
            "parts": [
                {
                    "type": "text",
                    "text": "What's the exchange rate for JPY to GBP today?"
                }
            ]
        },
        "acceptedOutputModes": ["text/plain"],
        "historyLength": 5
    }
}
```

- **Security:** Inter-Agent Communication (A2A):Inter-Agent Communication is a crucial aspect of system architecture, facilitating seamless and secure data exchange between various agents within the environment. To ensure robustness and integrity, A2A incorporates built-in mechanisms such as mutual Transport Layer Security (TLS), which establishes encrypted and authenticated connections between agents, preventing unauthorized access and data interception. Furthermore, A2A maintains comprehensive audit logs that meticulously record all inter-agent communications, providing detailed insights into the flow of information, the involved agents, and the specific actions performed. This audit trail is essential for accountability, troubleshooting, and security analysis, enabling administrators to track and trace communication events effectively. Authentication requirements for inter-agent communication are explicitly declared within the Agent Card, a configuration artifact that outlines the agent's identity, capabilities, and security policies. This centralized declaration ensures consistency and simplifies the management of authentication across the system. Agents typically authenticate themselves using credentials such as OAuth 2.0 tokens or API keys, which are securely passed via HTTP headers during the communication process. This method avoids the risk of credentials being exposed in the URL or message body, enhancing the overall security posture.

### A2A vs. MCP

A2A serves as a complementary protocol to Anthropic's Model Context Protocol (MCP) (see Fig. 1). While MCP concentrates on structuring context for agents and their interaction with external data and tools, A2A addresses the coordination and communication among agents, facilitating task delegation and collaboration. By establishing a standardized protocol for agent cooperation, A2A aims to increase efficiency, decrease integration expenses, and promote innovation and interoperability in the creation of complex, multi-agent AI systems. Understanding the fundamental elements and operational procedures of A2A is crucial for the effective design, implementation, and application of A2A in building interoperable and collaborative AI agent systems.

![Comparison A2A and MCP Protocols](./imgs/unnamed-22.png)
_Fig.1: Comparison A2A and MCP Protocols_

## Practical Applications & Use Cases

Inter-Agent Communication is indispensable for building sophisticated AI solutions across diverse domains, enabling modularity, scalability, and enhanced intelligence.

- **Multi-Framework Collaboration:** A2A's primary use case is enabling independent AI agents, regardless of their underlying frameworks (e.g., ADK, LangChain, CrewAI), to communicate and collaborate. This is fundamental for building complex multi-agent systems where different agents specialize in different aspects of a problem.
- **Automated Workflow Orchestration:** In enterprise settings, A2A can facilitate complex workflows by enabling agents to delegate and coordinate tasks. For instance, an agent might handle initial data collection, then delegate to another agent for analysis, and finally to a third for report generation, all communicating via the A2A protocol.
- **Dynamic Information Retrieval:** Agents can communicate to retrieve and exchange real-time information. A primary agent might request live market data from a specialized "data fetching agent," which then uses external APIs to gather the information and send it back.

## Hands-On Code Example

Let's examine the practical applications of the A2A protocol. The repository at [https://github.com/google-a2a/a2a-samples/tree/main/samples](https://github.com/google-a2a/a2a-samples/tree/main/samples) provides examples in Java, Go, and Python that illustrate how various agent frameworks, such as LangGraph, CrewAI, Azure AI Foundry, and AG2, can communicate using A2A. All code in this repository is released under the Apache 2.0 license. To further illustrate A2A's core concepts, we will review code excerpts focusing on setting up an A2A Server using an ADK-based agent with Google-authenticated tools. Looking at [https://github.com/google-a2a/a2a-samples/blob/main/samples/python/agents/birthday_planner_adk/calendar_agent/adk_agent.py](https://github.com/google-a2a/a2a-samples/blob/main/samples/python/agents/birthday_planner_adk/calendar_agent/adk_agent.py)

```python
import datetime
from google.adk.agents import LlmAgent # type: ignore[import-untyped]
from google.adk.tools.google_api_tool import CalendarToolset # type: ignore[import-untyped]

async def create_agent(client_id, client_secret) -> LlmAgent:
    """Constructs the ADK agent."""
    toolset = CalendarToolset(client_id=client_id, client_secret=client_secret)
    return LlmAgent(
        model='gemini-2.0-flash-001',
        name='calendar_agent',
        description="An agent that can help manage a user's calendar",
        instruction=f"""
        You are an agent that can help manage a user's calendar.
        Users will request information about the state of their calendar or to make changes to
        their calendar. Use the provided tools for interacting with the calendar API.
        If not specified, assume the calendar the user wants is the 'primary' calendar.
        When using the Calendar API tools, use well-formed RFC3339 timestamps.
        Today is {datetime.datetime.now()}.
        """,
        tools=await toolset.get_tools(),
    )
```

This Python code defines an asynchronous function `create_agent` that constructs an ADK `LlmAgent`. It begins by initializing a `CalendarToolset` using provided client credentials to access the Google Calendar API. Subsequently, an `LlmAgent` instance is created, configured with a specified Gemini model, a descriptive name, and instructions for managing a user's calendar. The agent is furnished with calendar tools from the `CalendarToolset`, enabling it to interact with the Calendar API and respond to user queries regarding calendar states or modifications. The agent's instructions dynamically incorporate the current date for temporal context. To illustrate how an agent is constructed, let's examine a key section from the `calendar_agent` found in the A2A samples on GitHub. The code below shows how the agent is defined with its specific instructions and tools. Please note that only the code required to explain this functionality is shown; you can access the complete file here: [https://github.com/google-a2a/a2a-samples/blob/main/samples/python/agents/birthday_planner_adk/calendar_agent/main.py](https://github.com/google-a2a/a2a-samples/blob/main/samples/python/agents/birthday_planner_adk/calendar_agent/main.py)

```python
def main(host: str, port: int):
    # Verify an API key is set.
    # Not required if using Vertex AI APIs.
    if os.getenv('GOOGLE_GENAI_USE_VERTEXAI') != 'TRUE' and not os.getenv(
        'GOOGLE_API_KEY'
    ):
        raise ValueError(
            'GOOGLE_API_KEY environment variable not set and '
            'GOOGLE_GENAI_USE_VERTEXAI is not TRUE.'
        )

    skill = AgentSkill(
        id='check_availability',
        name='Check Availability',
        description="Checks a user's availability for a time using their Google Calendar",
        tags=['calendar'],
        examples=['Am I free from 10am to 11am tomorrow?'],
    )

    agent_card = AgentCard(
        name='Calendar Agent',
        description="An agent that can manage a user's calendar",
        url=f'http://{host}:{port}/',
        version='1.0.0',
        defaultInputModes=['text'],
        defaultOutputModes=['text'],
        capabilities=AgentCapabilities(streaming=True),
        skills=[skill],
    )

    adk_agent = asyncio.run(create_agent(
        client_id=os.getenv('GOOGLE_CLIENT_ID'),
        client_secret=os.getenv('GOOGLE_CLIENT_SECRET'),
    ))

    runner = Runner(
        app_name=agent_card.name,
        agent=adk_agent,
        artifact_service=InMemoryArtifactService(),
        session_service=InMemorySessionService(),
        memory_service=InMemoryMemoryService(),
    )

    agent_executor = ADKAgentExecutor(runner, agent_card)

    async def handle_auth(request: Request) -> PlainTextResponse:
        await agent_executor.on_auth_callback(
            str(request.query_params.get('state')), str(request.url)
        )
        return PlainTextResponse('Authentication successful.')

    request_handler = DefaultRequestHandler(
        agent_executor=agent_executor, task_store=InMemoryTaskStore()
    )

    a2a_app = A2AStarletteApplication(
        agent_card=agent_card, http_handler=request_handler
    )

    routes = a2a_app.routes()
    routes.append(
        Route(
            path='/authenticate',
            methods=['GET'],
            endpoint=handle_auth,
        )
    )

    app = Starlette(routes=routes)
    uvicorn.run(app, host=host, port=port)

if __name__ == '__main__':
    main()
```

This Python code demonstrates setting up an A2A-compliant "Calendar Agent" for checking user availability using Google Calendar. It involves verifying API keys or Vertex AI configurations for authentication purposes. The agent's capabilities, including the "check_availability" skill, are defined within an `AgentCard`, which also specifies the agent's network address. Subsequently, an ADK agent is created, configured with in-memory services for managing artifacts, sessions, and memory. The code then initializes a Starlette web application, incorporates an authentication callback and the A2A protocol handler, and executes it using Uvicorn to expose the agent via HTTP.

These examples illustrate the process of building an A2A-compliant agent, from defining its capabilities to running it as a web service. By utilizing Agent Cards and ADK, developers can create interoperable AI agents capable of integrating with tools like Google Calendar. This practical approach demonstrates the application of A2A in establishing a multi-agent ecosystem.

Further exploration of A2A is recommended through the code demonstration at [https://www.trickle.so/blog/how-to-build-google-a2a-project](https://www.trickle.so/blog/how-to-build-google-a2a-project). Resources available at this link include sample A2A clients and servers in Python and JavaScript, multi-agent web applications, command-line interfaces, and example implementations for various agent frameworks.

## At a Glance

**What:** Individual AI agents, especially those built on different frameworks, often struggle with complex, multi-faceted problems on their own. The primary challenge is the lack of a common language or protocol that allows them to communicate and collaborate effectively. This isolation prevents the creation of sophisticated systems where multiple specialized agents can combine their unique skills to solve larger tasks. Without a standardized approach, integrating these disparate agents is costly, time-consuming, and hinders the development of more powerful, cohesive AI solutions.

**Why:** The Inter-Agent Communication (A2A) protocol provides an open, standardized solution for this problem. It is an HTTP-based protocol that enables interoperability, allowing distinct AI agents to coordinate, delegate tasks, and share information seamlessly, regardless of their underlying technology. A core component is the Agent Card, a digital identity file that describes an agent's capabilities, skills, and communication endpoints, facilitating discovery and interaction. A2A defines various interaction mechanisms, including synchronous and asynchronous communication, to support diverse use cases. By creating a universal standard for agent collaboration, A2A fosters a modular and scalable ecosystem for building complex, multi-agent Agentic systems.

**Rule of thumb:** Use this pattern when you need to orchestrate collaboration between two or more AI agents, especially if they are built using different frameworks (e.g., Google ADK, LangGraph, CrewAI). It is ideal for building complex, modular applications where specialized agents handle specific parts of a workflow, such as delegating data analysis to one agent and report generation to another. This pattern is also essential when an agent needs to dynamically discover and consume the capabilities of other agents to complete a task.

**Visual summary**

![A2A inter-agent communication pattern](./imgs/unnamed-23.png)
_A2A inter-agent communication pattern_

## Key Takeaways

Key Takeaways:

- The Google A2A protocol is an open, HTTP-based standard that facilitates communication and collaboration between AI agents built with different frameworks.
- An `AgentCard` serves as a digital identifier for an agent, allowing for automatic discovery and understanding of its capabilities by other agents.
- A2A offers both synchronous request-response interactions (using `tasks/send`) and streaming updates (using `tasks/sendSubscribe`) to accommodate varying communication needs.
- The protocol supports multi-turn conversations, including an `input-required` state, which allows agents to request additional information and maintain context during interactions.
- A2A encourages a modular architecture where specialized agents can operate independently on different ports, enabling system scalability and distribution.
- Tools such as Trickle AI aid in visualizing and tracking A2A communications, which helps developers monitor, debug, and optimize multi-agent systems.
- While A2A is a high-level protocol for managing tasks and workflows between different agents, the Model Context Protocol (MCP) provides a standardized interface for LLMs to interface with external resources

## Conclusions

The Google A2A protocol is an open, HTTP-based standard for communication between AI agents from different frameworks. It uses `AgentCards` as digital identities, detailing capabilities and authentication for automatic discovery and interaction. A2A supports synchronous request-response (`tasks/send`) and streaming updates (`tasks/sendSubscribe`), enabling various interaction patterns, including multi-turn dialogues. Its modular architecture allows agents to operate independently, enhancing scalability. A2A provides a standardized framework for collaborative AI, enabling agents to leverage each other's capabilities effectively.

## References

- Chen, B. (2025, April 22). How to Build Your First Google A2A Project: A Step-by-Step Tutorial. _Trickle.so Blog_. [https://www.trickle.so/blog/how-to-build-google-a2a-project](https://www.trickle.so/blog/how-to-build-google-a2a-project)
- Google A2A GitHub Repository. [https://github.com/google-a2a/A2A](https://github.com/google-a2a/A2A)
- Google Agent Development Kit (ADK) [https://google.github.io/adk-docs/](https://google.github.io/adk-docs/)
- Getting Started with Agent-to-Agent (A2A) Protocol: [https://codelabs.developers.google.com/intro-a2a-purchasing-concierge#0](https://codelabs.developers.google.com/intro-a2a-purchasing-concierge#0)
- Google AgentDiscovery - [https://google-a2a.github.io/A2A/topics/agent-discovery/](https://google-a2a.github.io/A2A/topics/agent-discovery/)
- Communication between different AI frameworks such as LangGraph, CrewAI, and Google ADK [https://www.trickle.so/blog/how-to-build-google-a2a-project](https://www.trickle.so/blog/how-to-build-google-a2a-project)
