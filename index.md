---
layout: default
title: What is MICA
nav_order: 1
description: "The Enterprise Level Agentic Solution"
permalink: /
---

MICA (Multiple Intelligent Conversational Agents) is designed to simplify the development of customer service bots. There are numerous agent frameworks available—such as AutoGen, CrewAI, LangChain, Amazon MAO, and Swarm—these frameworks offer high flexibility for constructing agents in general settings. However, they tend to be overly complex for professionals with limited programming experience.
These frameworks often emphasize the orchestration of multiple agents, but their designs are frequently buried in intricate Python code, lacking a clear, big picture. We argue that the core of an agent framework should center on the agents themselves. Approaches like Swarm’s minimalist design and CrewAI’s use of agent configuration files offer promising directions.  Here, we would like to take one step further by introducing MICA, an agent centric framework. 
MICA comprises four types of agents: KB, LLM, Flow, and Ensemble. As their names suggest, the KB Agent handles information retrieval and question-answering tasks, while the LLM Agent encodes business logic and workflows using natural language. In contrast, the Flow Agent allows traditional control flows through a domain-specific language. The Ensemble Agent orchestrates these agents and serve customers collectively. Here is a skeleton of an airline chatbot using MICA.

```yaml
Flight Cancel:
  type: llm agent
  args:
   - flight_number
  description: It can handle flight cancel requests.
  prompt: <Flight Cancel Policy>
Flight Change:
  type: llm agent
  args:
    - flight_number
  description: It can handle flight change requests.
  prompt: <Flight Change Policy>
  uses:
    - change_flight
Lost Baggage:
  type: llm agent
  args:
    - flight_number
    - address
  description: It can handle baggage lost requests.
  prompt: <Lost Baggage Policy>
Meta:
  type: ensemble agent
  contain:
    - Flight Cancel
    - Flight Change
    - Lost Baggage
main:
  steps:
    - call: Meta
```

As a comparison, the same implementation is available for [Swarm](https://github.com/openai/swarm/tree/main/examples/airline) and XYZ (link).

<details>
  <summary>Show the code</summary>
  <pre><code>
def transfer_to_flight_modification():
    return flight_modification

def transfer_to_flight_cancel():
    return flight_cancel

def transfer_to_flight_change():
    return flight_change

def transfer_to_lost_baggage():
    return lost_baggage

def transfer_to_triage():
    """Call this function when a user needs to be transferred to a different agent and a different policy.
    For instance, if a user is asking about a topic that is not handled by the current agent, call this function.
    """
    return triage_agent
  </code></pre>
</details>
