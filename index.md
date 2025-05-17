---
layout: default
title: What is ADL/MICA
nav_order: 1
description: "The Enterprise Level Agentic Solution"
permalink: /
---
After years of exploring ideal platforms for customer service chatbots, we have arrived at a solution that meets our expectations: [ADL](https://arxiv.org/abs/2504.14787) (Agent Declarative Language for chatbot specification) and [MICA](https://github.com/Mica-labs/MICA) (Multiple Intelligent Conversational Agents), which interprets and runs programs written in ADL.

Here is a skeleton of an airline service bot written in ADL. Its full implementation is available at [MICA](https://github.com/Mica-labs/MICA/tree/main/examples). 

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
  type: flow agent
  steps:
    - call: Meta
```

Please refer to [a Swarm implementation] (https://github.com/openai/swarm/tree/main/examples/airline) for comparison.
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


ADL comprises four types of agents: KB, LLM, Flow, and Ensemble. Please check the [full specification](https://arxiv.org/pdf/2504.14787) of ADL.  ADL can be viewed as a YAML version of OpenAI's [Swarm](https://github.com/openai/swarm) (and [OpenAI Agent SDK](https://platform.openai.com/docs/guides/agents-sdk)).  It  abstracts away implementation details, providing a succinct description of agents and their relationship, making systematic prompting engineering, testing and debugging possible in the future. If you like this design, please give us a [star](https://github.com/Mica-labs/MICA#staying-ahead). 

[Sirui Zeng](https://siruizeng011.github.io/), [Xifeng Yan](https://sites.cs.ucsb.edu/~xyan/)  
Computer Science, University of California at Santa Barbara
