---
layout: default
title: Flow Control
parent: Concepts
nav_order: 3
---

To implement Flow Agent's control logic and ensure that conversations proceed according to the predefined flows, MICA defines a complete set of flow structures similar to traditional programming languages. It is simple and easy to understand while being fully compatible with the YAML specification.  

Here is a Flow Agent example that includes condition structures. 
```yaml
shopping_flow:
  type: flow agent
  description: This is an agent that assists users in placing orders.
  args:
    - discount
    - is_new_customer
  steps:
    - begin
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
    - end

    - begin: get_discount
    - if: is_new_customer == True
      then:
        - set: 
            discount: 0.9
        - return: success, Load discount successful.
      else:
        - return: error, No discount applied.
    - end
```

All the process statements are written under `<flow_name>.steps`, including subflow, user, bot, conditional control, loop, and call.

## Subflow
A subflow is similar to a single Python function and consists of a series of steps. A subflow is enclosed by special keywords, `"begin"` and `"end"` (if there is only one subflow, `"begin"` and `"end"` can be omitted).  

You can also assign a unique label to the subflow by changing `begin` to `begin: <label>`.  

Note that when multiple subflows exist, you must call other subflows within the first subflow for them to execute. This is because, by default, Flow only executes the first subflow.

```yaml
greeting:
  type: flow agent
  steps:
    - begin
    - bot: What can I do for you?
    - call: goodbye
    - end
    
    - begin: goodbye
    - bot: Goodbye!
    - end
```

The example above includes two different ways to define a subflow. When the agent executes, MICA first locates the initial subflow and then sequentially executes the steps within it. Since there is a `call` statement, it will automatically invoke the `goodbye` subflow.

## Bot
`Bot` is a keyword that intuitively represents the agent's next response. Currently, `Bot` only supports text output. You can use a `Bot` step anywhere within a subflow and can also write multiple consecutive bot responses.  

Additionally, you can use `${[agent_name].<arg_name>}` to reference any argument value from any agent. If `agent_name` is omitted, it refers to an argument within the current agent.
```yaml
greeting:
  type: flow agent
  args:
    - user
  steps:
    - set:
        user: "Honored guest"
    - bot: Hello!
    - bot: Wish you a good mood, ${user}!
```

## User
`User` indicates that the flow should listen for the user's next input at this point. We simply use the string `"user"` to represent it.  

Here is an example:
```yaml
greeting:
  type: flow agent
  steps:
    - bot: What can I do for you?
    - user
    - bot: I'm willing to tell you what I can do.
```

This example means that after the bot says, **"What can I do for you?"**, it waits for the user's input. Then, regardless of what the user says (unless they want to exit the flow), it directly responds with **"I'm willing to tell you what I can do."**  

However, in practice, we usually need to respond conditionally based on the user's intent. Therefore, we need conditional statements.

## Condition
Conditional statements allow selecting the next action based on different conditions. The condition can be either the user's intent or the value of a parameter.  

You can refer to this example:
```yaml
book_restaurant:
  type: flow agent
  args:
    - restaurant_name
    - num_of_people
  steps:
    - bot: What can I do for you?
    - user
    - if: the user claims "What can you do"
      then: 
        - bot: I can assist with booking a restaurant. 
    - else if: restaurant_name != None or num_of_people != None
      then:
        - call: collect_booking_information
      else:
        - bot: Sorry, I cannot understand what you mean.
        - return: error, cannot understand user's intent
```
This is a basic `if` structure. Below are some key concepts:  

- `if` / `else if`: Conditional statements that support checking both user intent and argument values.  
  - When using the user's intent as a condition, you must use the format:  
    `the user claims "<One user input example>"`  
  - When using argument values as conditions, MICA supports the following checks:  
    - Checking for `None` values:  
      - `<arg_name> == None` or `<arg_name> != None`  
    - Checking for a specific value:  
      - `<arg_name> == <str>`  
    - Checking if a value matches a pattern:  
      - `re.match(<regular expression>, <arg_name>)`  
  - When multiple conditions exist, you can combine them using `and` or `or`.  

- `then`: Specifies the actions to execute when the `if` condition is `True`. This is a list where you can define multiple flow steps. After all steps are executed, if there is no `return` statement, execution continues with the next statement in the current subflow.  

- `else` *(optional)*: Specifies the actions to execute when the `if` condition is `False`. This is also a list where multiple flow steps can be defined and executed sequentially.  

- `return` *(optional)*:  
  - `<success/error>, [msg]`: Exits the current subflow with either a `succeed` or `error` status.  
  - `msg`: Describes the success or failure message to inform other agents about the current agent’s status.  
  - MICA supports two statuses:  
    - `succeed`: Indicates a normal exit.  
    - `error`: Indicates an abnormal exit. The `msg` can help other agents understand the agent’s state for better scheduling in the next steps.

## Set
Sometimes, you need to pass parameters or manually assign values to arguments. In such cases, you can use the `set` keyword.
```yaml
recommend_restaurant:
  type: flow agent
  args:
    - restaurant_name
    - restaurant_description
  steps:
    - user
    - if: the user claims "Recommend a fast-food restaurant."
      then: 
        - set:
            restaurant_name: "In-N-Out"
            restaurant_description: web_searcher.get_description
        - bot: Why not try ${restaurant_name}
```

Here, the `set` keyword can be followed by any argument of any agent. You can also assign the value of an argument from one agent to any argument of another agent.  

In the example above, we set two argument values:  
- `<arg_name>: <str>`: Assigns a specific value to this argument. If it's an argument from another agent, use `<agent_name>.<arg_name>`.  
- `<one_arg_name>: <another_arg_name>`: Assigns the value of another argument to this argument. Similarly, you can use `<agent_name>.<arg_name>` to reference an argument from any agent.

## Label & Next
Sometimes, we need to jump to another subflow for execution or return to an earlier point within a subflow. In such cases, we use the `label` keyword for positioning and `next` to specify the target.
```yaml
book_restaurant:
  type: flow agent
  steps:
    - label: back_to_search
    - call: search_recommend
    - if: search_recommend.error
      then:
      - bot: Something wrong with the searching service. We will try again.
      - next: back_to_search
        tries: 3
```
In this example: 
- `label`: Defines a label name. Within the same flow, label names should be unique to avoid conflicts.
- `next`: Specifies the label to jump to next. This label can be defined in any subflow of the agent or as the subflow's label (defined using `begin: <label>`).  
- `tries` *(optional)*: Specifies the maximum number of times this statement can be executed. If not specified, it defaults to unlimited retries.

## Call
When you need to invoke any external agent or tool outside the current agent, you can use the `call` keyword to specify it.

Currently, **`call: <agent_name>`** supports the following types:  
- Tool 
- LLM agent  
- Flow agent

```yaml
tools:
  - search.py

book_restaurant:
  type: flow agent
  description: The user needs a restaurant recommendation and a table reservation.
  args:
    preference
  steps:
    - user
    - call: search_recommendation
      args:
        preference: preference
    - bot: "We found some restaurants you may like: ${search_recommend.restaurant}"
    - user
    - if: the user claims "Great. Can you help me make a reservation?"
    - call: collect_customers_info
```

Here, `search_recommendation` is a Python function. You need to implement the corresponding Python code and define the Python script name in the `tools` directory.  

Below is a simulated Python implementation. For more details on Python implementation guidelines, refer to [here](/docs/concepts/custom_function).

```python
import random
def search_recommendation(preference):
    # Mock restaurant database
    restaurants = {
        'Chinese': [
            {
                'name': 'Golden Dragon',
                'price_range': 'Average $35',
                'signature_dishes': ['Braised Pork Belly', 'Kung Pao Chicken', 'Dan Dan Noodles']
            },
            {
                'name': 'Canton House',
                'price_range': 'Average $45',
                'signature_dishes': ['Steamed Sea Bass', 'Salt & Pepper Shrimp', 'Crab Roe Tofu']
            }
        ],
        'Japanese': [
            {
                'name': 'Sakura Sushi',
                'price_range': 'Average $65',
                'signature_dishes': ['Premium Sashimi Platter', 'Torched Salmon Nigiri', 'Uni Sushi']
            },
            {
                'name': 'Yamada Kitchen',
                'price_range': 'Average $40',
                'signature_dishes': ['Tempura Set', 'Udon Noodles', 'Tonkatsu']
            }
        ],
        'Western': [
            {
                'name': 'La Maison',
                'price_range': 'Average $60',
                'signature_dishes': ['Beef Wellington', 'Lobster Bisque', 'Tiramisu']
            },
            {
                'name': 'Olive Garden',
                'price_range': 'Average $35',
                'signature_dishes': ['Fettuccine Alfredo', 'Grilled Lamb Chops', 'Caesar Salad']
            }
        ]
    }
    
    
    
    try:
        # Check if the requested cuisine type exists
        if preference not in restaurants:
            return {
                'error': f'Sorry, no recommendations found for {preference} cuisine'
            }
        
        # Randomly select a restaurant matching the preference
        restaurant = random.choice(restaurants[preference])
        
        return [{"slot_name": "restaurant", "value": {
            'name': restaurant['name'],
            'price_range': restaurant['price_range'],
            'signature_dishes': restaurant['signature_dishes']
        }}]
    
    except Exception as e:
        return {
            'error': f'Error during recommendation process: {str(e)}'
        }
```

In the example above, the second `call: collect_customers_info` is an LLM agent, which means you need to define this agent in the same `agents.yml` file.

```yaml
collect_customers_info:
  type: llm agent
  description: This is an agent used to collect restaurant reservation information.
  args:
    - number_of_people
    - outdoor
    - time
    - preferences
  prompt: |
    You are a bot that collects users' restaurant reservation information. 
    You need to collect the following details in order: number of people, indoor or outdoor seating, 
    reservation time, and any other preferences.
```
