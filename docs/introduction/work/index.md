---
layout: default
title: How MICA Works
nav_order: 3
---

In real-world service conversations, users may articulate their needs in an unpredictable order. MICA is specifically designed to handle this complexity efficiently. We conceptualize a service bot as a collection of autonomous agents, each capable of independently determining the next course of action—whether to switch agents, wait for user input, conclude the conversation, or take other necessary steps. Ultimately, our goal is to delegate as much conversational control as possible to the LLM, ensuring a more flexible and natural user experience, though control can still be regained by Flow Agents whenever necessary. 

<center>
<img style="width: 70%; height: auto;" src="schedule.png">
<br>
<div> Agent Scheduling in MICA </div>
</center>

The diagram shows a single-turn conversation. Our chatbot is essentially a pool of agents, with the state of this pool managed by the ensemble agent. When there is a user input, the ensemble agent, through the controller, analyzes and selects the most relevant agent from the pool to respond. This agent then takes over the conversation, meaning that as long as the agent believes it can handle the user’s conversation, it will continue to respond to the user. This assumption is based on the observation that users, in general, do not frequently switch topics, but rather continue within the current context. There is no need to frequently reassess which agent is the most appropriate one to respond.  In its future release, MICA will provide other scheduling options. 

When an agent clearly determines that the user has changed his intension, it will release control, handing it back to the ensemble agent, which will then choose the next agent to respond. This process continues until the agent determines that it is time for the user to speak again.

With this mechanism, MICA ensures the independence and modularity of individual agents while minimizing the need to call the language model for triaging.  This reduces cost and latency.
