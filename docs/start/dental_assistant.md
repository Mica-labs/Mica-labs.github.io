---
layout: default
title: "Dental Assistant Chatbot"
parent: Quick Start
nav_order: 6
---

This example demonstrates most of the core features of ADL through a dental assistant chatbot that supports patient record creation, appointment scheduling, patient information retrieval, dental image analysis, and SMS notifications. The source code is available [here](https://github.com/Mica-labs/MICA/tree/main/examples/third_party/dental_assistant).

The **PatientRecord** agent gathers the patient’s full name, contact information, and date of birth, then calls a backend function to create a new patient record in the system.

The **AppointmentScheduler** agent assists with booking dental appointments. It collects the patient’s name, asks for a preferred date in a specific format, checks availability, schedules the appointment if possible, and optionally routes to **NotificationSender** to send a confirmation SMS.

The **PatientInformationInquiry** agent looks up and returns personal and appointment details for an existing patient, using either their name or ID.

The **ImageAnalysis** agent handles dental image or X-ray review requests by passing the provided image identifier to a backend analysis function.

The **NotificationSender** agent sends appointment reminders or other messages to patients via SMS, using Twilio.


```yaml
# agents.yml
PatientRecord:
  type: llm agent
  description: Creates a brand-new patient record by gathering name, contact info, and DOB.
  prompt: |
    1. Greet the user and confirm they want to create a new patient record.
    2. Ask for the patient's full name.
    3. Ask for contact information (phone or email).
    4. Ask for the patient's date of birth in **MM/DD/YYYY** format (e.g. 08/24/1990).
    5. Call `action_create_patient_record(name, contact_info, date_of_birth)`.
    6. Confirm record creation to the user.
  args:
    - name
    - contact_info
    - date_of_birth
  uses:
    - action_create_patient_record


AppointmentScheduler:
  type: llm agent
  description: Guides the user through scheduling a dental appointment.
  prompt: |
    1. Greet the user and confirm they want to schedule an appointment.
    2. Ask for the patient's name.
    3. Ask for the desired appointment **date only**, and instruct:
       “Please enter the date in **MM/DD/YYYY** format (e.g. 07/06/2025).”
    4. Call `action_check_availability(appointment_datetime)`.
    5. If available, call `action_schedule_appointment(name, appointment_datetime)` and confirm.
    6. After confirming: “Your appointment is scheduled for <date>, <name>.
       Would you like me to send a confirmation SMS to the patient?” 
       — If the user says “yes”, the **meta** router will dispatch to **NotificationSender**.
    6. If not available, apologize and ask for a different date (again reminding them of the MM/DD/YYYY format).
  args:
    - name
    - appointment_datetime
  uses:
    - action_check_availability
    - action_schedule_appointment



PatientInformationInquiry:
  type: llm agent
  description: Looks up an existing patient's personal and appointment info.
  prompt: |
    1. Confirm the user wants to look up a patient's information.
    2. Ask for the patient's full name **or** ID.
    3. Call `action_get_patient_info(name, patient_id)`.
    4. Present the returned information to the user.
  args:
    - name
    - patient_id
    - contact_info
    - date_of_birth
    - appointments
  uses:
    - action_get_patient_info



ImageAnalysis:
  type: llm agent
  description: Handles X-ray or image analysis requests.
  prompt: |
    1. Confirm the user wants an image analyzed.
    2. Ask for the image identifier or URL.
    3. Call `action_analyze_image(image_id)`.
    4. Present the analysis results to the user.
  args:
    - image_id
  uses:
    - action_analyze_image


NotificationSender:
  type: llm agent
  description: Sends appointment reminders or other notifications via SMS.
  prompt: |
    1. Confirm the user wants to send a notification.
    2. Ask for the recipient's phone number.
    3. Ask for the message text.
    4. Call `action_send_notification(recipient, message)
    5. Confirm the notification was sent.
  args:
    - sender_number
    - recipient
    - message
  uses:
    - action_send_notification


meta:
  type: ensemble agent
  description: Routes user requests to the appropriate sub-agent.
  contains:
    - PatientRecord
    - AppointmentScheduler
    - PatientInformationInquiry
    - ImageAnalysis
    - NotificationSender
  fallback: default
  exit: default
  steps:
    - bot: "Hello! I am your dental assistant. How can I help you today?"

main:
  type: flow agent
  steps:
    - call: meta
````

Below is a simple implementation of the custom functions involved. You can also connect your own database or implement any custom logic you need.

````python
#tools.py
import json
from pathlib import Path
import sqlite3
from os import getenv
from dotenv import load_dotenv
from openai import OpenAI
from twilio.rest import Client as TwilioClient


load_dotenv()
client = OpenAI(api_key=getenv("OPENAI_API_KEY"))

twilio_client = TwilioClient(
    username=getenv("TWILIO_ACCOUNT_SID"),
    password=getenv("TWILIO_AUTH_TOKEN")
)

# initialize the database
conn = sqlite3.connect("dental.db")
cur = conn.cursor()
cur.execute("""
CREATE TABLE IF NOT EXISTS patients (
    id INTEGER PRIMARY KEY,
    name TEXT,
    contact TEXT,
    dob TEXT
)
""")
cur.execute("""
CREATE TABLE IF NOT EXISTS appointments (
    id INTEGER PRIMARY KEY,
    patient_id INTEGER,
    datetime TEXT,
    FOREIGN KEY(patient_id) REFERENCES patients(id)
)
""")



conn.commit()
conn.close()

# --- Tool functions for MICA agents ---

def get_conn():
    """Open a connection to the SQLite database."""
    return sqlite3.connect("dental.db")


def action_create_patient_record(name, contact_info=None, date_of_birth=None):
    """Create DB row, emit tool JSON, then fill slots and speak once."""
    conn = get_conn()
    cur = conn.cursor()
    cur.execute(
        "INSERT INTO patients(name, contact, dob) VALUES (?,?,?)",
        (name, contact_info, date_of_birth)
    )
    conn.commit()
    patient_id = cur.lastrowid
    conn.close()

    tool_payload = {
        "patient_id": patient_id,
        "name": name,
        "contact_info": contact_info,
        "date_of_birth": date_of_birth
    }
    print(json.dumps(tool_payload))

    return [
        {"arg": "name",          "value": name},
        {"arg": "contact_info",  "value": contact_info},
        {"arg": "date_of_birth", "value": date_of_birth},
        {
            "bot": f" Patient record #{patient_id} created for {name}.",
            "status": "success"
        }
    ]


def action_check_availability(appointment_datetime, patient_id=None):
    """Return a JSON object indicating whether the time slot is free."""
    conn = get_conn()
    cur = conn.cursor()
    cur.execute(
        "SELECT COUNT(*) FROM appointments WHERE datetime = ?",
        (appointment_datetime,)
    )
    booked_count = cur.fetchone()[0]
    conn.close()

    result = {
        "available": (booked_count == 0)
    }
    print(json.dumps(result))
    return result
  

def action_schedule_appointment(name=None, appointment_datetime=None):
    """Schedule an appointment for an existing patient."""
    conn = get_conn()
    cur = conn.cursor()
    cur.execute("SELECT id FROM patients WHERE name = ?", (name,))
    row = cur.fetchone()
    if not row:
        conn.close()
        result = {
            "error": f"No patient named '{name}' found. Please create a record first."
        }
        print(json.dumps(result))
        return result

    patient_id = row[0]
    cur.execute(
        "INSERT INTO appointments(patient_id, datetime) VALUES (?, ?)",
        (patient_id, appointment_datetime)
    )
    conn.commit()
    appt_id = cur.lastrowid
    conn.close()

    result = {
      "data": {
        "name": name,
        "appointment_datetime": appointment_datetime
      },
      "appointment_id": appt_id,
      "text": f"Appointment #{appt_id} scheduled for {name} on {appointment_datetime}."
    }
    print(json.dumps(result))
    return result



def action_get_patient_info(name=None, patient_id=None):
    """Fetch info, fill slots, and reply."""
    conn = get_conn()
    cur = conn.cursor()

    if name:
        cur.execute("SELECT id, name, contact, dob FROM patients WHERE name = ?", (name,))
    elif patient_id:
        cur.execute("SELECT id, name, contact, dob FROM patients WHERE id = ?", (patient_id,))
    else:
        return [{"bot": "Please provide the patient's name or ID.", "status": "error"}]

    row = cur.fetchone()
    if not row:
        who = name or patient_id
        return [{"bot": f"No patient found matching '{who}'.", "status": "error"}]

    pid, pname, contact, dob = row
    cur.execute("SELECT id, datetime FROM appointments WHERE patient_id = ?", (pid,))
    appts = [{"appointment_id": aid, "datetime": dt} for aid, dt in cur.fetchall()]
    conn.close()

    result = {
        "patient_id": pid,
        "name": pname,
        "contact": contact,
        "dob": dob,
        "appointments": appts
    }

    print(json.dumps(result))

    return [
        {"arg": "patient_id: ", "value": pid},
        {"arg": "name",       "value": pname},
        {"arg": "contact_info","value": contact},
        {"arg": "date_of_birth","value": dob},
        {"arg": "appointments","value": appts},
        {"status": "success"}
    ]


def action_analyze_image(image_id):
    """
    Download the image from the given URL, send it to gpt-4o with vision support,
    and return exactly one BotUtter event so the analysis appears only once.
    """
    prompt = "Please provide a very brief analysis and diagnosis of this dental image."

    try:
        chat_completion = client.chat.completions.create(
            model="gpt-4o",
            max_tokens=300,
            messages=[
                {
                    "role": "user",
                    "content": [
                        {"type": "text",      "text": prompt},
                        {"type": "image_url", "image_url": {"url": image_id}}
                    ]
                }
            ]
        )
        analysis = chat_completion.choices[0].message.content.strip()
    except Exception as e:
        analysis = f"Error during image analysis: {e}"

    return [
        {
            "bot": analysis,
            "status": "success"
        }
    ]

def action_send_notification(
        sender_number,       # <- ask for / store this once
        recipient,
        message,
):
    """
    Send an SMS to `recipient`
    with text `message`.  Prints JSON: {"sid": "...", "status": "sent"}
    so the agent can echo it verbatim.
    """
    try:
        msg = twilio_client.messages.create(
            body=message,
            from_=sender_number,
            to=recipient,
        )
        result = {"sid": msg.sid, "status": "sent"}
    except Exception as e:
        result = {"error": str(e)}
    print(json.dumps(result))
    return result
````

