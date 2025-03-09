---
layout: default
title: Custom Function
parent: Concepts
nav_order: 2
---

MICA includes an internal sandbox that can execute custom Python code. You can implement your own validation functions, webhooks, or database access.

You need to write custom function in a `.py` file and include the `.py` file in the agent yml through `tools:` The names of arguments in custom function should be meaningful, ideally matching the argument names defined in the agents. The language model relies on this mechanism to find corresponding arguments.

## Function Structure

You can execute your own function code within an LLM agent or a flow agent.
1. Using an LLM agent as an example
Suppose you want to implement an LLM agent for querying the weather. The LLM agent will ask the user for the location and date they want to check, then call a function that queries a weather API to retrieve real-time weather information. Finally, the LLM agent will generate a natural language response based on the function’s result.
To achieve this, you need to specify in the prompt when to call the function and declare the function name under uses when defining the LLM agent. Below is the agents.yml definition for the weather-querying LLM agent:
```yaml
tools:
  - tools.py
Weather Inquiry:
  type: llm agent
  description: Can handle user requests for weather queries
  prompt: |
    You need to ask the user for the location and date they want to check, then call "get_weather_info".
  uses:
    - get_weather_info
```

Next, you need to implement get_weather_info() in tools.py. Below is an example:
```python
import requests

def get_weather_info(location, date):
    """
    Queries WeatherAPI to retrieve weather information for a specified location and date.

    :param location: City name or coordinates (e.g., "New York", "37.7749,-122.4194")
    :param date: Query date in "YYYY-MM-DD" format
    :param api_key: Your WeatherAPI access key
    :return: A dictionary containing weather information
    """
    api_key = "your key here"
    base_url = "http://api.weatherapi.com/v1/history.json"
    params = {
        "key": api_key,
        "q": location,
        "dt": date
    }

    try:
        response = requests.get(base_url, params=params)
        response.raise_for_status()  # Raise HTTPError if the request fails
        print(response.text)  # The LLM will process this output internally
        return [{'text': 'I have retrieved the weather for you.'}, {'slot_name': 'weather_info', 'value': response.json()}]
    except requests.exceptions.RequestException as e:
        print('Query failed. Reason:', e)
        return
```


The example above uses a public API to query historical weather data for a given location and date.
In an actual conversation, the LLM agent will ask the user for location and date, then internally request execution of get_weather_info, passing the collected parameters to this function.

Once executed, all print() statements will be passed back to the LLM, which will analyze them to generate the next utterance. The user will not see any content printed by print().
If you want this function to directly respond to the user, you need to structure the return output in a specific format.

2. Manually Defining Function Calls Using Call Step.
In addition to LLM function calls, you can manually define function calls at any point during the conversation (steps can exist in main/ensemble agent/flow agent). In certain workflows, at the beginning or end of a conversation, you may need to interact with external sources via functions.

For example, if you need to retrieve user information from a database before the conversation starts, you must first specify a function call in the steps of the ensemble agent. This is the content of agents.yml:
```yaml
meta:
  type: ensemble agent
  contains:
    - book_restaurant
    - order_online
  args:
    - username
    - email
    - address
  steps:
    - call: get_info
      args:
        username: "Amy"
    - set:
        email: get_info.email
        address: get_info.address
```

Correspondingly, in tools.py, you need to implement get_info():
```python
def get_info(username):
    db = Database()
    query = """
                SELECT name, email, address
                FROM users
                WHERE name = ?
            """
    result = db.run_query(query, (username,), one_record=True)

    if result:
        username, email, address = result
        return [
            {"slot_name": "email", "value": email},
            {"slot_name": "address", "value": address},
        ]
    else:
        return
```


In the return of this function, the value is assigned to the args of get_info. Using the set statement, the values can be copied to the ensemble agent. If other agents have args with the same name, they can share the value.

Here is a complete function implementation template:
```python
def function_name(required_arg1, required_arg2, optional_arg1="value1", optional_arg2="value2"):
    """function description (optional)"""
    # your logic here
    print("execution result")   # If this function is requested by the LLM, you can only communicate with the LLM via stdout (print statements)
    return [{"text": "chatbot reply here."}, 
            {"slot_name": "updated_arg", "value": "updated_value"}]     # If you want to directly add a bot response or modify some returned values, you should return them in this format
```


## Scenario
### Argument Validation
Here is an example of an argument validation function that checks if the parameter `amount_of_money` is a positive number.
```python
def validate_money(amount_of_money):
    money = float(amount_of_money)
    if money > 0:
        print("amount_of_money is a positive number")
    else:
        print("amount_of_money is not a positive number")
    return
```
Since all arguments are treated as strings by default in MICA, they shall be converted to the required data type for further processing.

### Web Query
When you need to interact with an external API, you can implement a function similar to the one below. The parameters can be arguments defined within the bot. Meanwhile, MICA supports passing the response from the request back to the bot. 

Note that the content of `print()` will not be used as the bot's response. It is only transmitted to GPT as the execution result when needed. If you want to generate a bot response based on the web query's response, you need to include this response in the `return` statement. 

Additionally, if you need to update the value of certain arguments, you can return the argument name and its updated value in the specified format.

```python
import requests
def web_query(arg1, arg2):
    method = ["POST", "GET", "PUT", "DELETE", "PATCH"]  # Choose according to your webhook API.

    headers = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36'
    }
    url = "http://target_url"
    params = {"arg1": arg1, "arg2": arg2}
    timeout = 30
    verify = True
    
    request_kwargs = {
        'url': url,
        'params': params,
        'headers': headers,
        'timeout': timeout,
        'verify': verify
    }
    
    try:
        response = requests.request(method, **request_kwargs)
        response.raise_for_status()
        
        # The content printed to stdout will not be displayed in the bot but will be used as a response to GPT when GPT calls tools.
        print(response.text) 
        
        if response.json():
            # You can pass external information back to the bot as an argument value.
            return [{"slot_name": "<agent_name>.<arg_name>", "value": response.json().get("updated_arg")}] 
        
        # If it is a string, it will be used as the bot's response.
        return response.text  
    except requests.RequestException as e:
        print(f"Request failed: {str(e)}")
        raise
```

### Connect to Database
Another common scenario is connecting to a database to validate or store argument values. The following example demonstrates the Python code required for a simple transfer bot.

The function `connect_db()` establishes a connection to an `sqlite3` database. The function `validate_account_funds(amount_of_money)` checks whether the account balance is greater than the transfer amount. Finally, `submit_transaction(amount_of_money, recipient)` executes the transfer operation in the database.
```python
import sqlite3

def connect_db():
    return sqlite3.connect('user_info.db')


def validate_account_funds(amount_of_money):
    conn = connect_db()
    cursor = conn.cursor()

    cursor.execute("SELECT account_balance FROM user_info WHERE user_name = ?", ('user',)) 
    account_balance = cursor.fetchone()

    if account_balance is None:
        print("doesn't exist!")
        conn.close()
        return False

    if account_balance[0] >= amount_of_money:
        print("suffient")
        conn.close()
        return True
    else:
        print("insuffient")
        conn.close()
        return False

def submit_transaction(amount_of_money, recipient):
    conn = connect_db()
    cursor = conn.cursor()

    cursor.execute('''
    CREATE TABLE IF NOT EXISTS transactions (
        transaction_id INTEGER PRIMARY KEY AUTOINCREMENT,
        amount_of_money REAL,
        recipient TEXT,
        transaction_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
    ''')

    cursor.execute('''
    INSERT INTO transactions (amount_of_money, recipient)
    VALUES (?, ?)
    ''', (amount_of_money, recipient))

    conn.commit()
    conn.close()

    print(f"Success. Money: {amount_of_money}, recipient: {recipient}")
```
