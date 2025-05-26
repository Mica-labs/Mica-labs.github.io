---
layout: default
title: "Book Store Chatbot"
parent: Quick Start
nav_order: 5
---

This example demonstrates most of the core features of ADL through a bookstore chatbot that supports return policy inquiries, online ordering, and personalized book recommendations. The source code is available [here](https://github.com/Mica-labs/MICA/tree/dev/examples/bookstore).

The **store_policy_kb** agent handles questions about return policies by retrieving answers from internal documents, the website, and frequently asked questions. Whenever a user’s concern matches available information, the agent provides the relevant response.

The **book_recommendation** agent interacts with users to learn about their reading preferences. It searches the book database by genre and suggests top-selling titles that match the user’s interests.

Finally, the **order** agent manages the purchasing process. It confirms the items the user wants to buy and places the order. If the user expresses interest in exploring more options, this agent will call on the book_recommendation agent to continue the recommendation flow.

```yaml
store_policy_kb:
  type: kb agent
  description: I can answer questions related to the store’s policy.
  sources:
    - https://www.bookstores.com/policies/shipping
    - https://www.bookstores.com/about
  faq:
    - q: Thanks, bye
      a: Looking forward to serving you next time.  
      
book_recommendation:
  type: llm agent
  description: I can recommend books to customers.
  args:
    - genre
    - book_name
    - book_info
    - book_wanted
  prompt: |
    1. Ask the user if they have a preferred book genre.
    2. If the user has a favorite book, call the "query_book_genre" function based on their favorite book to obtain the "genre".
    3. Using the genre, call the "find_bestsellers" function to recommend relevant books to the user. Then ask user if they need to order this one.
    4. If the user agree, append this book to argument "book_wanted", which should be an array, and then complete this agent.
  uses:
    - query_book_genre
    - find_bestsellers

order:
  type: flow agent
  description: I can place an order.
  args:
    - books
    - order_status
  fallback: "Sorry, I didn’t understand that. Could you rephrase it?"
  steps:
    - bot: "I’ll place the order for you."
    - label: confirm_books
    - bot: "You have selected these books so far: ${books}. Would you like to add anything else to your order?"
    - user
    - if: the user claims "Yeah", "Do you have other types of books?"
      then:
        - call: book_recommendation
        - next: confirm_books
    - else if: the user claims "No", "I don’t have anything else I want to buy."
      then:
        - next: start_ordering_operation
      else:
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
    - book
  contains:
    - store_policy_kb
    - book_recommendation:
        args:
          book_wanted: ref book
    - order:
        args:
          books: ref book
  steps:
    - call: get_date
    - set:
        date: get_date.today
    - bot: "Hi, I’m your bookstore assistant. How can I help you?"
  exit: default

main:
  type: flow agent
  steps:
    - call: triage
```
Below is a simple implementation of the custom functions involved. You can also connect your own database or implement any custom logic you need.
```python
from datetime import date
import random

books = {
    "The Time Traveler's Diary": {
        "genre": "Fiction",
        "description": "A man discovers a journal that allows him to revisit moments in history and rewrite fate.",
        "rating": 4.5,
    },
    "Quantum Realities": {
        "genre": "Science",
        "description": "An accessible explanation of quantum physics and its implications for reality.",
        "rating": 4.2,
    },
    "The Dragon's Pact": {
        "genre": "Fantasy",
        "description": "A young mage forms a forbidden alliance with a dragon to save her crumbling kingdom.",
        "rating": 4.7,
    },
    "Deep Work": {
        "genre": "Non-fiction",
        "description": "Guidance on how to focus in a distracted world and achieve meaningful productivity.",
        "rating": 4.6,
    },
    "The Memory Garden": {
        "genre": "Fiction",
        "description": "An elderly woman tends a magical garden where forgotten memories bloom anew.",
        "rating": 4.4,
    },
    "AI and the Future of Thinking": {
        "genre": "Science",
        "description": "Explores how artificial intelligence is transforming the way humans reason and solve problems.",
        "rating": 4.3,
    },
    "The Last Enchanter": {
        "genre": "Fantasy",
        "description": "An orphan discovers he’s the final heir to an ancient magical order and must fight dark forces.",
        "rating": 4.6,
    },
    "Atomic Habits": {
        "genre": "Non-fiction",
        "description": "A proven system for building good habits and breaking bad ones.",
        "rating": 4.9,
    },
    "The Paper City": {
        "genre": "Fiction",
        "description": "In a crumbling metropolis made of paper, a journalist uncovers a conspiracy that could ignite a revolution.",
        "rating": 4.3,
    },
    "The Moonstone Tower": {
        "genre": "Fantasy",
        "description": "A knight seeks a legendary tower that grants wishes—but only at a cost.",
        "rating": 4.1,
    },
}

def get_date():
    today = date.today()
    formatted = today.strftime("%Y-%m-%d")
    return [{"status": "success"},
           {"arg": "today", "value": formatted}]

def query_book_genre(book_name):
    genres = ["Fiction", "Non-fiction", "Fantasy", "Science"]
    genre = random.choice(genres)
    if books.get(book_name):
        genre = books.get(book_name)['genre']
    print(f"The genre is : {genre}")
    return [{"arg": "genre", "value": genre}]

def find_bestsellers(genre = None):
    """If a specific genre is given, the function returns the best-selling book description in that genre; 
    otherwise, it returns the overall best-seller.
    """
    cand = []
    bestseller = {}
    if genre is not None:
        for name, info in books.items():
            if info["genre"].lower() == genre.strip().lower():
                cand.append({"name": name, "info": info})
    else:
        for name, info in books.items():
            cand.append({"name": name, "info": info})
    sorted_cand = sorted(cand, key=lambda x: x["info"]["rating"], reverse=True)
    bestseller = sorted_cand[0]
    print(f"The bestseller is: {bestseller['name']}, the description: {bestseller['info']['description']}")
    return [{"arg": "book_info", "value": bestseller['info']['description']},
           {"arg": "book_name", "value": bestseller["name"]}]
  
def place_order(ordered_book, date):
    return [{"bot": "Okay, your order is all set!"},
           {"arg": "status", "value": True}]
```