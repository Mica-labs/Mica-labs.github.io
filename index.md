---
layout: default
title: What is MICA
nav_order: 1
description: "The Enterprise Level Agentic Solution"
permalink: /
---

[MICA](https://github.com/Mica-labs/MICA) (Multiple Intelligent Conversational Agents) is now live on GitHub and available as open source. After years of exploring the ideal architecture for customer service bots, we finally arrived at a solution that we are satisfied with. It is a kind of YAML version of OpenAI's [Swarm](https://github.com/openai/swarm) (and [OpenAI Agent SDK](https://platform.openai.com/docs/guides/agents-sdk)).  However, it has an SQL-like mindset. Mica abstracts away implementation details, providing a succinct description of agents and their relationship, making systematic prompting engineering, testing and debugging possible in the future. 

There are numerous agent frameworks available—such as [AutoGen](https://github.com/microsoft/autogen), [CrewAI](https://github.com/crewAIInc/crewAI), [LangChain](https://github.com/langchain-ai/langchain), [Amazon MAO](https://github.com/awslabs/multi-agent-orchestrator), and [Swarm](https://github.com/openai/swarm) —these frameworks offer high flexibility for constructing agents in general settings. However, they tend to be overly complex for professionals in the customer service domain who have limited programming experience.
While emphasizing the orchestration of multiple agents, their designs are frequently buried in intricate Python code, lacking a clear, big picture. We argue that the core of an agent framework should center on the agents themselves. Approaches like Swarm’s minimalist design and CrewAI’s use of agent configuration files offer promising directions.  MICA takes a bold step forward by placing natural language programming of agents at the core of the framework.

Here is a skeleton of an airline service bot written in MICA. Its full implementation is available [here](https://github.com/Mica-labs/MICA/tree/main/examples). If you like this design, please give us a [star](https://github.com/Mica-labs/MICA#staying-ahead).  Your support will motivate us to continue adding new features, including tracing, testing, and debugging.

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
  contains:
    - Flight Cancel
    - Flight Change
    - Lost Baggage

main:
  steps:
    - call: Meta
```

As a comparison, the Swarm implementation is [here](https://github.com/openai/swarm/tree/main/examples/airline).

<details>
  <summary>Partial Implementation of Swarm</summary>
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

def triage_instructions(context_variables):
    customer_context = context_variables.get("customer_context", None)
    flight_context = context_variables.get("flight_context", None)
    return f"""You are to triage a users request, and call a tool to transfer to the right intent.
    Once you are ready to transfer to the right intent, call the tool to transfer to the right intent.
    You dont need to know specifics, just the topic of the request.
    When you need more information to triage the request to an agent, ask a direct question without explaining why you're asking it.
    Do not share your thought process with the user! Do not make unreasonable assumptions on behalf of user.
    The customer context is here: {customer_context}, and flight context is here: {flight_context}"""

triage_agent = Agent(
    name="Triage Agent",
    instructions=triage_instructions,
    functions=[transfer_to_flight_modification, transfer_to_lost_baggage],
)

flight_modification = Agent(
    name="Flight Modification Agent",
    instructions="""You are a Flight Modification Agent for a customer service airlines company.
      You are an expert customer service agent deciding which sub intent the user should be referred to.
You already know the intent is for flight modification related question. First, look at message history and see if you can determine if the user wants to cancel or change their flight.
Ask user clarifying questions until you know whether or not it is a cancel request or change flight request. Once you know, call the appropriate transfer function. Either ask clarifying questions, or call one of your functions, every time.""",
    functions=[transfer_to_flight_cancel, transfer_to_flight_change],
    parallel_tool_calls=False,
)

flight_cancel = Agent(
    name="Flight cancel traversal",
    instructions=STARTER_PROMPT + FLIGHT_CANCELLATION_POLICY,
    functions=[
        escalate_to_agent,
        initiate_refund,
        initiate_flight_credits,
        transfer_to_triage,
        case_resolved,
    ],
)

flight_change = Agent(
    name="Flight change traversal",
    instructions=STARTER_PROMPT + FLIGHT_CHANGE_POLICY,
    functions=[
        escalate_to_agent,
        change_flight,
        valid_to_change_flight,
        transfer_to_triage,
        case_resolved,
    ],
)

lost_baggage = Agent(
    name="Lost baggage traversal",
    instructions=STARTER_PROMPT + LOST_BAGGAGE_POLICY,
    functions=[
        escalate_to_agent,
        initiate_baggage_search,
        transfer_to_triage,
        case_resolved,
    ],
)
  </code></pre>
</details>

MICA comprises four types of agents: KB, LLM, Flow, and Ensemble. As their names suggest, KB Agents handle information retrieval and question-answering tasks, while LLM Agents deal with business logic and workflows using natural language. In contrast, Flow Agents allow traditional control flows through a domain-specific language. An Ensemble Agent orchestrates these agents and selects the right agent to respond.  That is all.  MICA's principle is to minimize the introduction of new concepts.

[Sirui Zeng](https://siruizeng011.github.io/), [Xifeng Yan](https://sites.cs.ucsb.edu/~xyan/)  
Computer Science, University of California at Santa Barbara
