---
layout: default
title: Example
nav_order: 4
---

The [Rasa Starter Pack for Retail Banking](https://github.com/rasa-customers/starterpack-retail-banking-en/tree/main) is an example project that demonstrates how to build a conversational assistant for banking services using Rasa. This assistant can help users transfer money, check balances, manage payees, and block cards.
对于每个agent，Rasa需要定义每一个需要收集的变量，以及flow的完整流程。我会在下面详细介绍这一点。

We have successfully implemented the same features of this Rasa example using MICA. You can find the complete implementation [here](https://github.com/Mica-labs/MICA/tree/main/examples/retail_banking), or use our provided visual interface to load and test it directly. Below, we outline key features from the repository and compare our implementation with MICA, highlighting how MICA simplifies the process compared to Rasa's more complex setup.

## Feature Comparison

### 1. **Money Transfer**
This function requires the user to first verify whether the payee exists, then collect a series of variables, and finally complete the transfer. In MICA, you can use the following LLM agent to implement this process. Here is a part of `agents.yml`:
```yaml
transfer_money:
  type: llm agent
  description: Guides the user through the process of initiating a bank transfer.
  prompt: |  
    1.	First, confirm that you understand the user wants to transfer money.
    2.	Ask for the username. Based on the username, call the function “action_ask_account_from” to display the account name and balance. Ask which account to use and fill in account_from with the corresponding number (do not show the account number to the user, just populate account_from directly).
    3.	Ask for payee_name. Using account_from and payee_name, call the function “action_check_payee_existence”.
    4a. If the payee does not exist, prompt the user to enter the “add payee” agent and exit this agent.
    4b. If the payee exists, continue collecting the transfer amount amount. If amount ≤ 0, prompt that the amount is invalid.
    5.	Once all required details are collected, call “action_check_sufficient_funds”.
    6a. If there are sufficient funds, proceed to step 7.
    6b. If there are insufficient funds, terminate this agent immediately.
    7.	Collect the transfer timing timing.
    8a. If timing is “now”, confirm whether to proceed with the immediate transfer.
    	•	If confirmed, call “action_process_immediate_payment”, then output “Transfer successful”, and exit this agent.
    	•	If not proceeding immediately, output “Transfer canceled”, and exit this agent.
    8b. If timing is not “now”, ask for the specific transfer date, ensuring it is formatted as “DD/MM/YYYY”. Proceed to step 9.
    9.	Call “action_validate_payment_date”.
    	•	If the date is a future date, confirm whether to schedule the transfer.
    	•	If confirmed, call “action_schedule_payment”, output “Transfer scheduled”, and exit this agent.
    	•	If not scheduling, output “Transfer canceled”, and exit this agent.
  args:
    - username
    - account_from
    - payee_name
    - amount
    - timing
    - payment_date
  uses:
    - action_check_payee_existence
    - action_check_sufficient_funds
    - action_process_immediate_payment
    - action_validate_payment_date
    - action_schedule_payment
```

You also need to implement all the Python functions mentioned in the `uses` field in `tools.py`. Here, `action_check_payee_existence` is used as an example.
```python
def action_check_payee_existence(username, payee_name):
    db = Database()
    user_query = "SELECT id FROM users WHERE name = ?"
    user_result = db.run_query(user_query, (username,), one_record=True)
    if user_result is None:
        print("The user does not exist.")
        return
    user_id = user_result[0]

    check_payee_query = "SELECT id FROM payees WHERE user_id = ? AND name = ?"
    payee_result = db.run_query(
        check_payee_query, (user_id, payee_name), one_record=True
    )

    if payee_result:
        print("The payee exists.")
        return
    print("The payee does not exist.")
    return [{"text": f"{payee_name} is not an authorised payee. Let's add them!"}]
```
<details>
  <summary>Show the RASA Implementation</summary>
  <pre><code>
flows:
  transfer_money:
    description: Guides the user through the process of initiating a bank transfer.
    steps:
      - action: utter_transfer_money_understand
      - collect: account_from
      - collect: payee_name
      - action: action_check_payee_existence
        next:
          - if: not slots.payee_exists
            then:
              - call: add_payee
                next: "get_transfer_amount"
          - else: "get_transfer_amount"
      - id: "get_transfer_amount"
        collect: amount
        description: The amount of money to transfer
        rejections:
          - if: slots.amount <= 0
            utter: utter_invalid_amount
      - action: action_check_sufficient_funds
        next:
          - if: slots.sufficient_funds
            then: "get_payment_timing"
          - else:
              - action: utter_insufficient_funds
                next: END
      - id: "get_payment_timing"
        collect: timing
        next:
          - if: slots.timing == "now"
            then: "confirm_immediate_transfer"
          - else: "get_payment_date"
      - id: "confirm_immediate_transfer"
        collect: confirm_immediate_payment
        ask_before_filling: true
        next:
          - if: slots.confirm_immediate_payment
            then:
              - action: action_process_immediate_payment
              - action: utter_transfer_successful
                next: END
          - else: "transfer_cancelled"
      - id: "get_payment_date"
        collect: payment_date
        description: the future payment date of the money transfer. Convert date to DD/MM/YYYY format
      - action: action_validate_payment_date
        next:
          - if: slots.future_payment_date
            then: "confirm_future_payment"
          - else:
              - action: utter_past_payment_date
              - set_slots:
                - payment_date: null
                next: "get_payment_date"
      - id: "confirm_future_payment"
        collect: confirm_future_payment
        ask_before_filling: true
        next:
          - if: slots.confirm_future_payment
            then:
              - action: action_schedule_payment
              - action: utter_payment_scheduled
                next: END
          - else: "transfer_cancelled"
      - id: "transfer_cancelled"
        action: utter_cancel_transfer

slots:
  account_from:
    type: text
  payee_name:
    type: text
  payee_exists:
    type: bool
  amount:
    type: float
  sufficient_funds:
    type: bool
  timing:
    type: categorical
    values:
      - now
      - future
  payment_date:
    type: text
  confirm_immediate_payment:
    type: bool
  confirm_future_payment:
    type: bool
  payment_processed:
    type: bool
  payment_scheduled:
    type: bool
  valid_payment_date:
    type: bool
  future_payment_date:
    type: bool

responses:
  utter_transfer_money_understand:
    - text: "Okay lets transfer money"
      metadata:
        rephrase: True
  utter_ask_payee_name:
    - text: "Which payee would you like to send money to?"
  utter_ask_amount:
    - text: "How much money would you like to transfer?"
  utter_insufficient_funds:
    - text: "I'm sorry, but you have insufficient funds for this transfer. Please enter a different amount"
  utter_ask_timing:
    - text: "When would you like this transfer to be made?"
      buttons:
        - title: Immediate
          payload: "/SetSlots(timing=now)"
        - title: Future
          payload: "/SetSlots(timing=future)"
  utter_ask_payment_date:
    - text: "On which date would you like this payment to be made?"
  utter_ask_confirm_future_payment:
    - text: "A payment of ${amount} to {payee_name} will be scheduled for {payment_date}. Is this correct?"
  utter_payment_scheduled:
    - text: "Your payment of ${amount} to {payee_name} has been successfully scheduled for {payment_date}"
  utter_ask_confirm_immediate_payment:
    - text: "An immediate payment of ${amount} to {payee_name} will be processed. Is this correct?"
  utter_transfer_successful:
    - text: "Your transfer of ${amount} to {payee_name} has been successfully processed"
  utter_cancel_transfer:
    - text: "No problem. I will cancel this transfer"
  utter_invalid_amount:
    - text: "You have to enter an amount greater than 0"
      metadata:
        rephrase: True
  utter_past_payment_date:
    - text: "A future payment date cannot be in the past!"
      metadata:
        rephrase: True

actions:
  - action_ask_account_from
  - action_check_payee_existence
  - action_check_sufficient_funds
  - action_schedule_payment
  - action_process_immediate_payment
  - action_validate_payment_date

</code></pre>
</details>

### 2. **Card Blocking**
When a user requests to block a card, it is necessary to ask for the reason and adopt different response strategies based on the given reason. Here is our LLM agent design for handling this process.
```yaml
block_card:
  type: llm agent
  description: ask for reason and block the card
  prompt: |
    You are an agent assisting users in blocking their cards.
    1. Based on the username, directly call "action_ask_card" to retrieve the user's available cards. Ask the user:  
       "Select the card you require assistance with:" and fill in the "card" field.  
    2. Ask the user for the reason they want to block the card (reason_for_blocking):  
       - The reason should be described as lost, damaged, stolen, suspected of fraud, malfunctioning, or expired.  
       - The user may also say they are traveling or moving, or that they want to temporarily freeze their card.  
       - For any other responses, set the "reason_for_blocking" slot to "unknown".  
    3a. If "reason_for_blocking" is "damaged" or "expired", inform the user:  
       "Thank you for letting us know. I'm sorry to hear the card was {reason_for_blocking}." 
       Proceed to Step 4.  
    3b. If "reason_for_blocking" is "fraud", "stolen", or "lost", inform the user:  
       "As your card was potentially stolen, it's crucial to report this incident to the authorities. Please contact your local law enforcement agency immediately."  
       Proceed to Step 4.  
    3c. If "reason_for_blocking" is "traveling" or "moving", inform the user:  
       "Thanks for informing us about moving."  
       Proceed to Step 4.  
    3d. For all other cases, instruct the user to contact "020 7777 7777" and call the function "action_update_card_status", then end the agent process.  
    4. Inform the user that their card will be blocked due to the specified reason.  
       - Confirm whether they want to **issue a new card** or if they prefer to visit the bank themselves.  
       - If the user confirms they want to issue a new card, proceed to Step 5.  
       - If the user declines, call "action_update_card_status" and end the agent process.  
    5. Confirm the user's "physical_address".  
       - If the user confirms the address is correct, inform them:  
         "The new card will be sent to this address."  
       - If the user indicates the address is incorrect, go back to Step 3d.  

  args:
    - username
    - card
    - physical_address
    - reason_for_blocking
  uses:
    - action_ask_card
    - action_update_card_status
```
<details>
  <summary>Show the RASA Implementation</summary>
  <pre><code>
flows:
  block_card:
    description: "Block or freeze a user's debit or credit card to prevent unauthorized use, stop transactions, or report it lost, stolen, damaged, or misplaced for added security"
    name: block a card
    steps:
      - action: utter_block_card_understand
      - call: select_card
      - collect: reason_for_blocking
        description: |
          The reason for freezing or blocking the card, described as lost, damaged, stolen, suspected of fraud,
          malfunctioning, or expired. The user may say they are traveling or moving, or they may say they want to
          temporarily freeze their card. For all other responses, set reason_for_blocking slot to 'unknown'.
        next:
          - if: "slots.reason_for_blocking == 'damaged' or slots.reason_for_blocking == 'expired'"
            then: "acknowledge_reason_damaged_expired"
          - if: "slots.reason_for_blocking == 'fraud' or slots.reason_for_blocking == 'stolen' or slots.reason_for_blocking == 'lost'"
            then:
              - set_slots:
                  - fraud_reported: true
                next: "acknowledge_reason_fraud_stolen_lost"
          - if: "slots.reason_for_blocking == 'traveling' or slots.reason_for_blocking == 'moving'"
            then:
              - set_slots:
                  - temp_block_card: true
                next: "acknowledge_reason_travelling_moving"
          - else: "contact_support"
      - id: acknowledge_reason_damaged_expired
        action: utter_acknowledge_reason_damaged_expired
        next: "confirm_issue_new_card"
      - id: acknowledge_reason_fraud_stolen_lost
        action: utter_acknowledge_reason_fraud_stolen_lost
        next: "card_blocked"
      - id: acknowledge_reason_travelling_moving
        action: utter_acknowledge_reason_travelling_moving
        next: "card_blocked"
      - id: "card_blocked"
        action: "utter_card_blocked"
        next: "confirm_issue_new_card"
      - id: "confirm_issue_new_card"
        collect: confirm_issue_new_card
        description: |
          Confirm if the user wants to be issued a new card. The answer should be an affirmative statement,
          such as "yes" or "correct," or a declined statement, such as "no" or "I don't want to"
        ask_before_filling: true
        next:
          - if: "slots.confirm_issue_new_card"
            then: "retrieve_user_address"
          - else: "update_card_status"
      - id: "retrieve_user_address"
        collect: address_confirmed
        description: |
          Confirm if the given address is correct. The answer should be an affirmative statement, such as "yes" or
          "correct," or a declined statement, such as "no" or "that's not right."
        next:
          - if: "slots.address_confirmed"
            then: "card_sent"
          - else: "contact_support"
      - id: "card_sent"
        action: utter_confirm_physical_address
        next: update_card_status
      - id: "contact_support"
        action: utter_contact_support
        next: update_card_status
      - id: "update_card_status"
        action: action_update_card_status
        next: END

slots:
  reason_for_blocking:
    type: categorical
    values:
      - lost
      - fraud
      - stolen
      - damaged
      - expired
      - traveling
      - moving
  address_confirmed:
    type: bool
  fraud_reported:
    type: bool
    initial_value: false
  temp_block_card:
    type: bool
    initial_value: false
  confirm_issue_new_card:
    type: bool
  address:
    type: text
  card_status:
    type: categorical
    values:
      - active
      - inactive
actions:
  - action_update_card_status

responses:
  utter_ask_reason_for_blocking:
    - text: "Please tell us the reason for blocking"
      buttons:
      - title: "I lost my card"
        payload: "/SetSlots(reason_for_blocking=lost)"
      - title: "My card is damaged"
        payload: "/SetSlots(reason_for_blocking=damaged)"
      - title: "I suspect fraud on my account"
        payload: "/SetSlots(reason_for_blocking=fraud)"
      - title: "My card has expired"
        payload: "/SetSlots(reason_for_blocking=expired)"
      - title: "I'm planning to travel soon"
        payload: "/SetSlots(reason_for_blocking=traveling)"
      - title: "I'm moving to a new address"
        payload: "/SetSlots(reason_for_blocking=moving)"
  utter_block_card_understand:
    - text: "Okay, we can block a card. Let's do it in a few steps"
      metadata:
        rephrase: True
  utter_ask_address_confirmed:
    - text: "I have found your address: {physical_address}. Should the new card be delivered there?"
      buttons:
        - title: "Yes"
          payload: "/SetSlots(address_confirmed=True)"
        - title: "No"
          payload: "/SetSlots(address_confirmed=False)"
  utter_confirm_physical_address:
    - text: "Your card will be delivered to {physical_address} within 7 business days"
  utter_card_blocked:
    - condition:
        - type: slot
          name: fraud_reported
          value: true
      text: "Since you have reported {reason_for_blocking}, we will block your card"
    - condition:
        - type: slot
          name: temp_block_card
          value: true
      text: "Since you are {reason_for_blocking}, we will temporarily block your card."
    - text: We will block your card.
  utter_ask_confirm_issue_new_card:
    - text: "Would you like to be issued a new card?"
      buttons:
        - title: "Yes, send me a new card"
          payload: "/SetSlots(confirm_issue_new_card=true)"
        - title: "No, just block my card"
          payload: "/SetSlots(confirm_issue_new_card=false)"
  utter_ask_address:
    - text: "Would you like us to deliver your new card to this address: {physical_address}?"
      buttons:
        - title: "Yes, send a new card"
          payload: "/SetSlots(address_confirmed=true)"
        - title: "No, I'll go to the bank"
          payload: "/SetSlots(address_confirmed=false)"
  utter_contact_support:
    - text: "Should you require further assistance, please contact our support team at 020 7777 7777. Thank you for being a valued customer."
    - text: "If you have any questions or concerns, please don't hesitate to reach out to our support team at 020 7777 7777. We're here to help."
    - text: "For additional support, please contact our customer service team at 020 7777 7777. Thank you for being a valued customer."
  utter_acknowledge_reason_damaged_expired:
    - text: "Thank you for letting us know. I'm sorry to hear the card was {reason_for_blocking}"
      metadata:
        rephrase: True
  utter_acknowledge_reason_fraud_stolen_lost:
    - text: "As your card was potentially stolen, it's crucial to report this incident to the authorities. Please contact your local law enforcement agency immediately."
    - text: "Given the unfortunate potential theft of your card, please report this incident to your local law enforcement agency. We'll work together to minimize the impact of this situation."
  utter_acknowledge_reason_travelling_moving:
    - text: Thanks for informing us about moving.
</code></pre>
</details>

## Conclusion

MICA provides an efficient and scalable alternative to the Rasa Starter Pack for Retail Banking. While Rasa requires extensive predefined intents, structured dialogues, and manual configurations, MICA simplifies implementation through its adaptive, context-aware, and streamlined approach. These improvements enable faster development and more natural interactions for customers.

