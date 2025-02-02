---
layout: default
title: Quick Start
nav_order: 5
has_children: false
---

Getting started with MICA is straightforward.  There are two options: One is through a local GUI frontend; the other is being deployed with docker image.

## Local GUI Frontend

You can design and test the bot through a local GUI, which requires Python 3.8 or higher.  Please execute the following command to install the required dependencies:
```bash
pip install -r requirement.txt
```
Set the OPENAI KEY.
```bash
export OPENAI_API_KEY=<your key>
```
Then, start the service:
```bash
python demo.py
```
You can visit `http://localhost:8760` and start to design.
![gui.png](gui.png)

## Docker Image

Please ensure your environment has Docker 20+ installed. It is recommended to allocate 2 CPU cores and 8GB of free memory. Start with the following command:

```bash
docker run --hostname Mica -d -v /root/data:/data -p 8080:80 --name Mica Mica/pack:latest
```
- `--hostname Mica`: Setting up the hostname will prevent the container's hostname from changing upon each restart.
- `-v /root/data:/data`: Specifies the persistent data storage directory. You can modify this path as needed. If not configured, data will be lost after the container restarts.
- `-p 8080:80`: Specifies the external port mapping. By default, the external port is set to 80. To use a different port (e.g., 8080), include this parameter.
- `--name Mica`: Assigns a name to the container for easier reference in Docker commands.
- Two image options are available: `Mica/pack:latest` and `Mica/aio:latest`. The former uses collectors directly from GitHub, while the latter integrates the latest collectors into the image, albeit with a larger size.

After the container is started, you can check the logs to see if the system is running:
```bash
docker logs Mica
```

When you see the following log, it means the system is running:
```bash
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
 System is ready! You can access the service as follows:
     URL: http://172.17.0.36
     User: xxx@xxxx
     Password: xxx

 Any question or suggestion, please reach out to xxx@xxx.us! Enjoy!!!
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
```

If you want to upgrade the system, you can use the following command:
```
#!/bin/bash

echo "Upgrade ..."
docker pull Mica/pack:latest
docker stop Mica
docker rm Mica
docker run --hostname Mica -d -v /root/data:/data -p 8080:80 --name Mica Mica/pack:latest
echo "Upgrade Done!"
exit
```

## First Example
Below is an implementation of a chatbot that can handle the money transfer service. You can copy/paste the following agent code into the same YAML document, e.g., such as `agents.yml`, and the python code into a single python file, e.g., 
 `tool.py`
 
### LLM Agent
The transfer service requires collecting two values: the recipient and the transfer amount. Therefore, you can introduce an LLM agent named “transfer_money” and specify the process in its prompt. Additionally, you can include  two parameters.
```yaml
transfer_money:
  type: llm agent
  description: This is an agent for a money transfer request.
  prompt: "You are a smart agent for handling transferring money request. When user ask for transferring money, it is necessary to sequentially collect the recipient's information and the transfer amount. Then, the function \"validate_account_funds\" should be called to check whether the account balance is sufficient to cover the transfer. If the balance is insufficient, it should return to the step of requesting the transfer amount. Finally, before proceeding with the transfer, confirm with the user whether the transfer should be made and then call \"submit_transaction\"."
  args:
    - recipient
    - amount_of_money
  uses:
    - validate_account_funds
    - submit_transaction
```
### Tool Use
To ensure that the transfer amount is less than the actual account balance, we need to use a function tool to perform argument validation. Below is a custom function that retrieves the balance from a database and then checks the transfer amount.
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
```
Once the transfer request is confirmed, we usually connect to a database to perform the actual transfer operation. Here, we implement a Python function to accomplish this task. The LLM agent will automatically call this function to submit the transaction information.
```python
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
### Ensemble Agent
You can define an Ensemble Agent to manage and coordinate the activation of multiple agents. In a simple scenario, such as a money transfer example with only one agent, an Ensemble Agent is not required. However, if your chatbot includes multiple agents that must handle various user intents, an Ensemble Agent becomes essential. It serves as a central entity that organizes and directs the interactions between agents. For instance, if there is a Knowledge Base (KB) agent responsible for answering user queries, the Ensemble Agent would ensure seamless coordination between it and other agents.
```yaml
meta:
  type: ensemble agent
  description: You can select an agent to respond to the user’s question.
  contain:
    - transfer_money
    - kb
  fallback: default_agent
  exit:
    - policy: "After 5 seconds, give a closure prompt: Is there anything else I can help you with?  After another 30 seconds, then leave."
```
### Which Agent to Start
As the `agent.yml` file contains multiple agents, it is necessary to designate an initial agent to start the process. The main agent serves this purpose, acting as the entry point for the chatbot. Its steps can invoke any agent; however, in most cases, it calls an Ensemble agent to coordinate interactions among different agents.
```yaml
main:
  steps:
    - call: meta
```
