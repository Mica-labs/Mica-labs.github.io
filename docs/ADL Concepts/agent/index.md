---
layout: default
title: Agent
parent: ADL Concepts
nav_order: 1
---
Agent is the basic building block in ADL. You can create different agents based on the tasks you would like to assign. There are four types of agents in ADL: KB, LLM, Flow, and Ensemble Agent. KB agents handle information retrieval and question-answering tasks, while LLM agents deal with business logic and workflows using natural language. In contrast, Flow agents allow traditional control flows through a domain-specific language. An Ensemble agent orchestrates these agents. In the current implmentation of MICA, KB and LLM agents cannot contain or call other agents.  Flow agents can call both KB and LLM agents.
<center>
<img style="width: 30%; height: auto;" src="structure.png">
<br>
<div>Types of Agents</div>
</center>

## KB Agent
A KB Agent is designed to handle KBQA (Knowledge Base Question Answering) tasks. If you have FAQ questions, documents, or websites, and want your chatbot to answer user questions based on them, you only need to declare a KB Agent. The agent will handle all the tehnical details for you, including vectorization, indexing, RAG, etc. Here is an example:

```yaml
kb:   # agent name
  type: kb agent
  faq:
    - q: Are you a robot
      a: Yes, I'm your virtual assistant.
    - q: Goodbye
      a: Thank you, bye.
  sources:
    - http://url
    - /path/to/kb/files
```

The agent name can be any string that complies with YAML. This KB Agent contains four attributes:

- `faq`: You can pair questions and their answers. Set each question in `q` attribute and its answer in `a` attribute.
- `sources`: You can list all relevant websites or files here. Our engine will automatically process the content. If it is a URL, MICA will crawl the websites and include their content in the knowledge base. You can also specify a directory and place files there. Files with `.doc`, `.pdf`, or `.csv` extensions are supported. The engine will extract and incorporate all content from the directory into the knowledge base.

Given a user utterance, the KB Agent will determine if it is a question and if it can be answered by the knowledge base it builds.

## LLM Agent
LLM Agents serve as the core building block for task-oriented conversations. It specifies domain knowledge and constraints through prompt programming. Additionally, LLM Agents can use tools and states to communicate.  The following example shows an LLM Agent to handle money transfer. 

```yaml
transfer_money:
  type: llm agent
  description: This is an agent for transfer money request.
  prompt: |
    You are a smart agent for handling transferring money request. When user ask for transferring money, 
    it is necessary to sequentially collect the recipient's information and the transfer amount. 
    Then, the function "validate_account_funds" should be called to check whether the account balance is sufficient to cover the transfer. 
    If the balance is insufficient, it should return to the step of requesting the transfer amount. 
    Finally, before proceeding with the transfer, confirm with the user whether the transfer should be made and then call "submit_transaction".
  args:
    - recipient
    - amount_of_money
  uses:
    - validate_account_funds
    - submit_transaction
```
A typical LLM Agent includes the following attributes:

- `description`: This field provides a brief explanation of the agent’s functionality. Based on the conversation context, it is used to determine whether this agent should respond or not.
- `prompt`: This field details how the agent will do the task.  If a certain function needs to be called during the process, the prompt should indicate when and which function to call.
- `args` (optional): If you need to pass or extract specific information (such as state, slot, variable, or any other relevant data) between the agent and others, you may populate this field. Please note that, at present, all variables are treated as strings.
- `uses` (optional): This lists the functions used in the prompt. These functions shall be provided in a separate Python file.

When calling an LLM Agent, MICA will automatically fill in all the args based on the defined content and call the corresponding Python functions. This process continues until the LLM Agent’s task is completed or the user changes his mind and needs a different agent to handle his request. 

## Flow Agent
Flow Agents are made for expressing fixed, sequential business logic.  It offers flow control similar to traditional programming languages, but in a YAML format.
```yaml
shopping_flow:
  type: flow agent
  description: This is an agent that assists users in placing orders.
  args:
    - discount
    - is_new_customer
  steps:
    - bot: Hi. I'm your shopping assistant. What can I do for you today?
    - label: start
    - user
    - if: the user claims "Is there any discount?"
      then:
        - next: get_discount
        - if: get_discount.success
          then:
            - bot: We are glad to tell you that your get 10% discount.
          else:
            - bot: Sorry. There's no discount for you.
    - else if: the user claims "I'd like to buy something"
      then:
        - call: order
        - call: shipment
        - call: submit_info
          args:
            - items: order.items
            - discount: discount
            - address: shipment.address
            - contact: shipment.contact
      else:
        - bot: "You can ask me something like \"Any discount?\" or \"Start shopping.\"."
        - next: start
          tries: 3

  get_discount:
    - if: is_new_customer == True
      then:
        - set: 
            discount: 0.9
        - return: success, Load discount successful.
      else:
        - return: error, No discount applied.

  fallback: "Give a fallback message: I'm sorry, I didn't understand that. Can you please rephrase? If fallback three times consecutively, then terminate the conversation."
```
A Flow Agent typically has the following attributes:

- `description`: Similar to LLM Agent, the description of the Flow Agent should briefly explain its functionality.
- `args` (optional): Similar to LLM Agent. 
- `steps`: This is the body of the Flow Agent, where all the logic is written.  Please refer to [Flow Control](/docs/ADL%20Concepts/flow_control) for more details. 
- `fallback` (optional): If the user’s input is unrelated to the current flow and this field is defined, the flow will follow the specified fallback policy. Otherwise, the flow will terminate immediately.

## Ensemble Agent
Ensemble Agents are different from the other three agents that have actual conversational functionality. Its main role is to coordinate different agents to provide responses.  Here is an example,
```yaml
Meta:
  type: ensemble agent
  contains:
    - flow_agent:
        args:
          flow_arg_name: ensemble_arg_name
    - llm_agent:
        args:
          llm_arg_name: ref ensemble_arg_name
    - kb_agent
  args:
    - user_name
    - user_gender
  fallback: default_agent
  exit: "exit policy here"
```
You need to fill out the following information:

- `contains`: List all the agents managed and coordinated by this Ensemble Agent.
- `args` (optional): Similar to LLM Agent. If the arg names defined here match those in the agents listed, the values from those agents will be automatically filled into this field.
- `fallback` (optional): If the user’s input cannot be handled by any agent listed and this field is defined, the flow will follow the specified fallback policy. Otherwise, there will be no response. You can also define a policy to describe the trigger condition of the fallback.
- `exit` (optional): When the user has completed a task and this field is defined,  the agent will terminate the conversation after 3 tries.  Otherwise, the agent will continue running.  You can customize your exit condition.
