---
layout: default
title: Why MICA
parent: Introduction
nav_order: 1
---

# Why MICA

---
\(informal version\)
## Why Do We Create MICA?
MICA, as the next-generation conversational chatbot design framework, has the following features.

### Effective Evaluation
Before deploying a chatbot, it is critical to evaluate its performance, response speed, and accuracy. In this regard, Swarm encourages developers to bring their own evaluation suites to test the performance of their swarms, which indicates that they lack an effective built-in evaluation tool. The examples they provide show a focus on single-turn messages. However, in practice, user intent often depends on context, requiring a holistic and comprehensive evaluation approach to assess the quality of conversations across various user intents.
An ideal evaluation tool should:
Understand the functions of all agents within a chatbot and their interconnections.
Generate comprehensive test cases to evaluate individual agents, collaborative agent performance, and unanticipated use cases.
Large language models (LLMs), with their strong language comprehension capabilities, are particularly suited for this task. Based on this, our YAML-format chatbot is better suited for implementing evaluation tools. Since our chatbot is well-organized, readable, and easy to understand, an LLM can fully comprehend its structure. In contrast, Swarm organizes agents in Python code without explicitly defining logical relationships between agents (e.g., whether two agents should execute sequentially or in parallel), potentially leading LLMs to overlook some paths.

### Comprehensive Design
Swarm defines the smallest unit of a chatbot as an "agent" without further classification. In contrast, MICA categorizes agents into four distinct types based on practical dialogue needs: Ensemble Agent, LLM Agent, Flow Agent and KB Agent. Each type has unique characteristics suited to handling different types of tasks, such as task-oriented dialogues, FAQs, or open-domain question answering.

### Flexibility and Collaboration
Our tool strictly follows the YAML specification, which aligns with human-readable formatting. Even individuals without a coding background can design chatbots directly. On the other hand, Swarm requires developers to be familiar with Python, making it less accessible to non-technical users.

### Support for Multiple LLM Models
Swarm is deeply integrated with OpenAI and heavily dependent on its functionality. In contrast, our YAML framework is model-agnostic, serving purely as a specification without mandating the use of a specific LLM. Our framework provides interfaces for third-party LLMs, allowing users to configure their own models through our APIs.


[Try and Explore MICA Today!](../start/) 

