---
layout: default
title: Agent
parent: Concepts
nav_order: 1
---

# Agents

{: .no_toc .header }

----
An agent is the foundation of building a chatbot. You can create different agents based on the characteristics of your chatbot to implement its logic. We categorize all agents into four types: KB Agent, LLM Agent, Flow Agent, and Ensemble Agent.

## KB Agent
A KB Agent is designed to handle KBQA (Knowledge Base Question Answering) tasks. If you have FAQ questions, documents, or websites, and want your chatbot to answer user questions based on this knowledge base, you need to define a KB Agent.

Here is an example of a KB Agent:

```yaml
kb:   # agent name
  type: KB agent
  faq:
    Are you a robot?: Yes, I'm your virtual assistant.
    Goodbye: Thank you, bye.
  web:
    - http://url
  file: /path/to/kb/files
```

The agent name can be any string that complies with YAML formatting. This KB Agent contains four attributes:

- `faq`: You can define specific questions and their corresponding answers. Use each question as a key and its answer as the value.
- `web`: You can list all relevant websites here. Our engine will automatically crawl the content of these websites, segment it into embeddings, and use it as part of the knowledge base.
- `file`: This field includes all files with `.doc`, `.pdf`, or `.csv` extensions. You can specify a directory and place all files in this path. The engine will read all the content in this directory and segment it into embeddings.

For each user input, the KB Agent will first perform embeddings, rank them by similarity, and select the most similar text segments as candidate answers. These candidates are then passed to other agents (e.g., Ensemble Agent) to determine whether the KB Agent's response can be used.

## LLM Agent
The LLM Agent is the most basic agent in task-oriented conversations. It excels at performing information-gathering tasks, such as checking the weather, making restaurant reservations, and so on.
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
The above is an example of how to use an LLM Agent to handle transfer tasks. A complete LLM Agent needs to include the following attributes:

- `description`: This field provides a brief explanation of the agent’s functionality. It serves as the description for the agent, and LLM will use this field to determine whether this agent should be used for the response.
- `prompt`: This field details the process for the agent. If certain functions need to be called during specific steps, the prompt should indicate which function to call.
- `args` (optional): If you need to record specific information, you should fill out this field. Currently, all args are strings.
- `uses` (optional): This lists all the function names used in the prompt. These functions can be implemented in a Python script.

When calling the LLM Agent, MICA will automatically fill in all the args based on the defined content and call the corresponding Python functions based on the LLM’s response. This process continues until the LLM Agent’s task is completed or the user changes their mind and requires a different agent to handle the response.

## Flow Agent
The Flow Agent is suitable for fixed, serialized business logic. We combine programming languages with YAML format to define various required logic.
```yaml
shopping_flow:
  type: flow agent
  description: This is an agent that assists users in placing orders.
  args:
    - discount
  steps:
    - begin
    - bot: Hi. I'm your shopping assistant. What can I do for you today?
    - label: start
    - user
    - if: the user claims "Is there any discount?"
      then:
        - call: discount
        - if: discount.success
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
            - user_name: meta.user_name
            - items: order.items
            - discount: discount
            - address: shipment.address
            - contact: shipment.contact
      else:
        - bot: "You can ask me something like \"Any discount?\" or \"Start shopping.\"."
        - call: start
          tries: 3
    - end

    - begin: discount
    - if: meta.is_new_customer == True
      then:
        - set: 
            discount: 0.9
        - return: success, Load discount successful.
      else:
        - return: error, No discount applied.
    - end
  fallback:
    - policy: "Give a fallback message: I'm sorry, I didn't understand that. Can you please rephrase? If fallback three times consecutively, then terminate the conversation."
```
A Flow Agent needs to include the following attributes:

- `description`: Similar to the LLM Agent, the description of the Flow Agent should briefly explain its functionality.
- `args` (optional): All args that need to be collected within the flow. Similar to the LLM Agent, if args are defined here, they will be automatically filled into the corresponding fields when a user step is defined in the flow.
- `steps`: This is the main attribute of the Flow Agent, where all the logic is written. I will explain the different types of steps in more detail below.
- `fallback` (optional): If the user’s input is unrelated to the current flow and this field is defined, the flow will follow the specified fallback policy. Otherwise, the flow will terminate immediately.

## Ensemble Agent
The Ensemble Agent is different from the previous agents that have actual conversational functionality. Its main role is to manage and assign different agents to handle responses.
```yaml
Meta:
  type: ensemble agent
  contain:
    - flow agent
    - llm agent
  args:
    - user_name
    - user_gender
  fallback: default_agent
  exit: 
    policy: "exit policy here"
```
Here is an Ensemble Agent, and you need to fill out the following information:

- `contain`: List all the agent names managed and scheduled by this Ensemble Agent.
- `args` (optional): Similar to before, the parameters for this Ensemble Agent are defined here. If the arg names defined here match those in the agents listed in contain, the arg values from those agents will be automatically filled into this field.
- `fallback` (optional): If the user’s input cannot be handled by any agent in contain, and no fallback is defined, there will be no response. Otherwise, you can specify any agent (LLM Agent/Flow Agent) to handle the fallback response. You can also define a policy to describe your fallback logic.
- `exit` (optional): When the user has completed a specific process, if exit is not defined, the bot will continue to wait. If defined, the bot will automatically end the conversation after 3 responses with no user input. You can also customize your exit policy, similar to how you define a fallback.