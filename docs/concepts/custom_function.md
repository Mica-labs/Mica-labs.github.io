---
layout: default
title: Custom Function
parent: Concepts
nav_order: 2
---

MICA includes an internal sandbox that can execute custom Python code. You can implement your own validation functions, webhooks, or database access.

You need to write custom function in a `.py` file and include the `.py` file in the agent yml through `tools:` The names of arguments in custom function should be meaningful, ideally matching the argument names defined in the agents. The language model relies on this mechanism to find corresponding arguments.  

## Argument Validation
Here is an example of an argument validation function that checks if the parameter `amount_of_money` is a positive number.
```python
def validate_money(amount_of_money):
    money = float(amount_of_money)
    return money > 0
```
Since all arguments are treated as strings by default in MICA,  they shall be converted to the required data type for further processing.

## Web Query
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

## Connect to Database
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
