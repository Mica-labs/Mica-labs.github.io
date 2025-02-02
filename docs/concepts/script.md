---
layout: default
title: Custom Function
parent: Concepts
nav_order: 2
---

MICA includes an internal sandbox that can execute custom Python code. You can implement your own validation functions, webhooks, or database access

You need to write custom function in a .py file and include the .py file in the agent yml through ``tools:'' The names of arguments in custom function should be meaningful, ideally matching the argument names defined in the agents. The language model relies on this mechanism to find corresponding arguments.  

## Argument Validation
Here is an example of an argument validation function that checks if the parameter amount_of_money is a positive number.
```python
def validate_money(amount_of_money):
    money = float(amount_of_money)
    return money > 0
```
Since all arguments are treated as strings by default in MICA,  they shall be converted to the required data type for further processing.

## Webhook
Please briefly explain what the following code is doing. 
```python
def web_query(**kwargs):
    url = "${url}"
    label = "${label}"
    headers = ${headers}
    params = ${request_body}
    text = "${request_body_text}"
    request_type = "${request_type}"
    request_body_type = "${request_body_type}"

    response_type = "${response_type}"
    response_handle = ${response_handle}
    default_error_msg = "抱歉，查询失败。"

    import requests
    import json
    import jsonpath
    from llmChatbot.event import BotUtter, SetSlot

    def request(url, params):
        response = None
        try:
            if request_body_type == "application/json":
                response = requests.request(request_type, url, data=json.dumps(params), headers=headers)
            else:
                response = requests.request(request_type, url, data=params, headers=headers)
        except Exception as e:
            print("Fail to get response from webhook. ", e)
        return response

    def __interpolator_text(response, states):
        import re
        try:
            text = re.sub(r"{([^\n{}]+?)}", r"{0[\1]}", response)
            text = text.format(states)
            if "0[" in text:
                # regex replaced tag but format did not replace
                # likely cause would be that tag name was enclosed
                # in double curly and format func simply escaped it.
                # we don't want to return {0[SLOTNAME]} thus
                # restoring original value with { being escaped.
                return response.format({})
            return text
        except Exception as e:
            print("cannot find states", e)
            return response

    def parse(json_dict, value):
        return jsonpath.jsonpath(json_dict, value)

    if request_body_type == "text/plain":
        text = __interpolator_text(text, kwargs)

    if request_body_type == "text/plain":
        response = request(url, text)
    else:
        response = request(url, params)
    result = []
    # 处理response
    if response is None or (response is not None and response.status_code >= 400):
        if response_handle is not None:
            result.append(BotUtter(response_handle.get("error_msg")))
        else:
            result.append(BotUtter(default_error_msg))


    if response_type == "direct":
        result.append(BotUtter(response.text))
        return result

    if response_type == "not":
        return []

    if response_handle is not None:
        if response is not None and response.status_code == 200:
            text = response_handle.get("text")
            for key, value in response_handle.get("parse").items():
                mapping = parse(response.json(), value)
                if not mapping:
                    result.append(BotUtter(response_handle.get("error_msg")))
                    break
                if len(mapping) == 0:
                    continue
                if len(mapping) == 1:
                    result.append(SetSlot(key, mapping[0]))
                else:
                    result.append(SetSlot(key, mapping))
            result.append(BotUtter(text))

        return result
```

## Connect to Database
Please briefly explain what the following code is doing. 
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
