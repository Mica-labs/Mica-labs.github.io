---
layout: default
title: "Book Store Chatbot"
parent: Quick Start
nav_order: 5
---

This example demonstrates most of the core features of ADL through a bookstore chatbot that supports return policy inquiries, online ordering, and personalized book recommendations.

The **store_policy_kb** agent handles questions about return policies by retrieving answers from internal documents, the website, and frequently asked questions. Whenever a user’s concern matches available information, the agent provides the relevant response.

The **book_recommendation** agent interacts with users to learn about their reading preferences. It searches the book database by genre and suggests top-selling titles that match the user’s interests.

Finally, the **order** agent manages the purchasing process. It confirms the items the user wants to buy and places the order. If the user expresses interest in exploring more options, this agent will call on the book_recommendation agent to continue the recommendation flow.

```yaml
store_policy_kb:
  type: kb agent
  description: I can answer questions related to the store’s policy.
  sources:
    - return_policy.pdf
    - www.bookstore.com/help-center
  faq:
    - q: Thanks, bye
      a: Looking forward to serving you next time.  
      
book_recommendation:
  type: llm agent
  description: I can recommend books to customers.
  args:
    - genre
    - book_info
  prompt: |
    1. Ask the user if they have a preferred book genre.
    2. If the user has a favorite book, call the "query_book_genre" function based on their 
    response to obtain the "genre".
    3. Using the genre and other attributes, call the "find_bestsellers" function to recommend relevant books to the user.
  uses:
    - query_book_genre
    - find_bestsellers

order:
  type: flow agent
  description: I can place an order.
  args:
    - books
    - order_status
  fallback: "Provide a fallback message: \"Sorry, I didn’t understand that. Could you rephrase it?\" If the fallback occurs three times consecutively, terminate the conversation."
  steps:
    - bot: "I’ll place the order for you."
    - label: confirm_books
    - bot: "You have selected these books so far: ${books}. Would you like to add anything else to your order?"
    - user
    - if: the user claims "Do you have other types of books?"
      then:
        - call: book_recommendation
        - next: confirm_books
    - else if: the user claims "I don’t have anything else I want to buy."
      then:
        - next: start_ordering_operation
      else:
        - call: store_policy_kb
        - next: confirm_books
          tries: 3
        
  start_ordering_operation:
    - call: place_order
      args:
        ordered_book: books
        date: triage.date
    - if: place_order.status == True
      then:
        - return: success, Order placed successfully.
      else:
        - return: error, Order failed.

triage:
  type: ensemble agent
  description: Select an agent to respond to the user.
  args:
    - date
  contains:
    - store_policy_kb
    - book_recommendation
    - order
  steps:
    - call: get_date
    - set:
        date: get_date.today
    - bot: "Hi, I’m your bookstore 
    assistant. How can I help you?"
  exit: default

main:
  type: flow agent
  steps:
    - call: triage
```