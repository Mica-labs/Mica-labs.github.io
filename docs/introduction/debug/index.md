---
layout: default
title: Debug ADL Programs
nav_order: 5
---

When developing a chatbot, various issues may arise. To support troubleshooting, MICA provides comprehensive logging capabilities for effective debugging.

## Using the Workbench
When programming with [MICA Workbench](/docs/start/index#mica-workbench), you can view agent logs directly. 
The figure below shows a log example from the [**bookstore example**](/docs/start/bookstore). The left side displays the actual conversation, while the right one shows the corresponding logs.

In this example, the user asks the bot to recommend a book. At this point:  
- The **store_policy_kb** agent generates a reply,  
- The **triage** component does not adopt it and instead assigns the task to the **book_recommendation** agent,  
- The **book_recommendation** agent then produces the next reply, which is adopted as the bot’s response.

<p align="center">
  <img src="img_3.png" alt="Image 1" width="48%">
  <img src="img_4.png" alt="Image 2" width="48%">
</p>

The log file provides a clear record of control transitions among the three agents. Furthermore, if an agent includes specific execution steps, the log identifies the step currently in progress. This enables you to verify that the chatbot is operating in alignment with your expectations.  
![img_1.png](img_1.png)

If you need more detailed logs, you can check the terminal where the service is running. There, you will find richer information, including **LLM request details**, which you can analyze to fine-tune your chatbot’s behavior. 

>The same information will also automatically be stored in the `logs` directory under the project root path, with the file name `<bot_name>.log`. When switching bots, the log will automatically be recorded in the corresponding log file.

![img_2.png](img_2.png)

All logs shown on the frontend page are at the **INFO** level, while the log file provides access to more detailed **DEBUG**-level information for further analysis. For readability, all intermediate messages in each dialogue round are indented by four spaces. 

## Deploying the Chatbot Service
When [deploying a chatbot](/docs/start/index#locally-deployment), you can control the log verbosity using different arguments.  

If you only need brief information, start the chatbot service with:  

```bash
python -m mica.server -v
```

If you need more detailed information, start it with:
```
python -m mica.server -vv
```

## Possible Types of Errors

We categorize potential issues in chatbot design into three main types:
1. **Prompt or agent design errors.** The chatbot behaves unexpectedly due to flaws in the prompt or agent design. In this case, analyze the logs, review the agent’s behavior, and adjust your chatbot’s logic.

2. **LLM response errors.** The LLM generates responses that do not align with your design. These issues can often be resolved by switching to a different model for the agent or applying prompt engineering techniques.

3. **Mica bugs.** If the logs suggest that the problem does not fall into the first two categories, it is likely a bug in Mica itself. In that case, we encourage you to open an issue on the [Mica repository](https://github.com/Mica-labs/MICA). We will respond and fix it promptly.
