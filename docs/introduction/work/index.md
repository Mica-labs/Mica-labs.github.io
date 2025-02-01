---
layout: default
title: How MICA Works
nav_order: 3
---

In real-world conversations, users may express their needs in any order.  MICA is designed to MICA is designed to effectively manage this complexity.  We conceptualize a chatbot as a collection of autonomous agents, each capable of independently determining the next course of action—whether to switch agents, wait for user input, conclude the conversation, or take other necessary steps. Ultimately, our goal is to delegate as much conversational control as possible to the LLM, ensuring a more flexible and natural user experience. On the other hand, control can be regained by Flow Agents whenever necessary. 

<center>
<img style="width: 70%; height: auto;" src="schedule.png">
<br>
<div> Agent Scheduling in MICA </div>
</center>

The diagram shows a single-turn conversation. Our chatbot is essentially a pool containing multiple agents, with the state of this pool managed by the ensemble agent. When there is a user input, the ensemble agent, through the controller, analyzes and selects the most relevant agent from the pool to respond. This agent then takes over the conversation, meaning that as long as the agent believes it can handle the user’s response, it will continue to respond to the user. This assumption is based on the observation that users, in general, do not frequently switch topics, but rather continue within the current context. There is no need to frequently reassess which agent is the most appropriate one to respond.  MICA will also provide other scheduling options in its future release. 

When an agent clearly determines that the user’s intention is no longer related to its task, it will release control, handing it back to the ensemble agent, which will then use the controller to choose the next agent to respond. This process continues until the agent determines that it is time for the user to speak again.

With this mechanism, MICA ensures the independence and modularity of individual agents while minimizing the need to call the LLM for triage.  This reduces costs and latency.
