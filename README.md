
# Python AI WhatsApp Bot (Disaster & Emergency Response)

An intelligent, real-time WhatsApp bot built with **Python**, **Flask**, the **Meta WhatsApp Cloud API**, and **Google Gemini 2.5 Flash**.

This system implements an automated crisis triage pipeline. When an individual texts `HELP`, a finite state machine (FSM) takes over to systematically gather location coordinates, injury reports, and headcounts. Upon completion, it dispatches incident notifications across multiple channels (Admin, Responders, and Dashboard APIs) while maintaining grounded conversational AI support using Google Gemini.

---

## Architecture & System Flow

```text
                             [ WhatsApp User ]
                                     │
                                     │ Inbound Webhook Event
                                     ▼
                        [ Meta WhatsApp Cloud API ]
                                     │
                                     │ HTTPS POST
                                     ▼
                          [ Flask App (run.py) ]
                                     │
                                     ▼
                        [ POST /webhook (views.py) ]
                                     │
                                     ▼
                          [ @signature_required ]
                         HMAC-SHA256 vs APP_SECRET
                                     │
                  ┌──────────────────┴──────────────────┐
                  │ Valid Signature?                    │
                  ├──────────────────┬──────────────────┤
                  │ No               │ Yes
                  ▼                  ▼
         [ HTTP 403 Forbidden ]  [ Inbound Filter ]
                                     │
                                     ├─ Status Updates (sent/delivered/read) ──► HTTP 200 (Drop)
                                     ├─ Seen Message IDs (Deduplication) ─────► Drop
                                     └─ Valid User Message
                                             │
                                             ▼
                                 [ Message Type Router ]
                                   (whatsapp_utils.py)
                                             │
               ┌─────────────────────────────┴─────────────────────────────┐
               │                                                           │
        [ msg_type == "text" ]                                   [ msg_type == "location" ]
               │                                                           │
        ┌──────┴──────────────────────┐                                    ▼
        │                             │                           Extract lat/lon pin
 [ "HELP" / Active Emergency ]  [ Idle / General ]                         │
        │                             │                           Update emergency record
        ▼                             ▼                           Prompt for injuries
[ Hardcoded FSM Steps ]       [ Gemini 2.5 Flash ]                         │
1. awaiting_location          (General Mode)                               ▼
2. awaiting_injury                    │                           [ Send via Meta API ]
3. awaiting_people_count              │
        │                             │
        ├─ Ongoing chat after intake? │
        │  └─► [ Gemini 2.5 Flash ]   │
        │      (Emergency Mode)       │
        │             │               │
        ▼             ▼               ▼
 [ Telemetry Complete ]       [ Send via Meta API ]
        │
        ├─────────────────────────────┬─────────────────────────────┐
        │                             │                             │
        ▼                             ▼                             ▼
[ External Dashboard ]        [ Admin & Responders ]         [ Victim Reply ]
HTTP POST to                  WhatsApp alerts via            "Responders alerted..."
DASHBOARD_API_URL             Meta Graph API                 via Meta Graph API

```

---

## Key Features

* **Automated Crisis State Machine**:
* Step-by-step state progression: `awaiting_location` → `awaiting_injury` → `awaiting_people_count` → `location_received`.
* Parses native WhatsApp GPS location objects (`latitude` & `longitude`) and unstructured text addresses (regex-based 6-digit Indian PIN codes and landmark keywords like *colony, nagar, road, sector, near*).
* NLP normalizers for injury detection (`yes`, `bleeding`, `hurt`, `no`, `fine`) and headcount extraction.


* **Dual-Persona AI via Google Gemini 2.5 Flash**:
* **Emergency Mode**: Keeps responses strictly to 1–2 short, reassuring sentences. System prompts explicitly forbid hallucinated rescue ETAs or responder claims.
* **General Mode**: Directs at-risk users to reply `HELP` and provides disaster helpline guidance.
* Context retention sliding window: Feeds the last 6 conversation turns into Gemini for situational continuity.


* **Multi-Tiered Alerting & Privacy Protection**:
* **Administrator (`ADMIN_NUMBER`)**: Receives the complete incident dossier, victim profile name, direct phone number, headcount, injury status, and an interactive Google Maps navigational pin.
* **Field Responders (`RESPONDER_NUMBERS`)**: Broadcasts a privacy-preserving alert to a comma-separated list of responder numbers directing them to check the central system without exposing victim PII.
* **Incident Dashboard (`DASHBOARD_API_URL`)**: Pushes an HTTP POST JSON payload to a centralized operations monitor.


* **Security & Reliability**:
* Validates all incoming payloads against `APP_SECRET` using `HMAC-SHA256` digest checks (`X-Hub-Signature-256`).
* Automated handshake verification for Meta's `GET /webhook` challenge.
* In-memory message ID deduplication (`_seen_message_ids`) prevents redundant processing on network retries.



---

## Tech Stack

* **Language**: Python 3.10+
* **Web Framework**: Flask (Blueprints, Application Factory)
* **AI / LLM Engine**: Google Generative AI SDK (`gemini-2.5-flash`)
* **API Integration**: Meta Graph API (WhatsApp Cloud API `v18.0`+)
* **Security**: HMAC SHA-256 (`hashlib`, `hmac`)
* **HTTP Client**: Requests

---

## Project Structure

```text
├── app/
│   ├── __init__.py                # Flask application factory (create_app)
│   ├── config.py                  # Configuration loader & logging configuration
│   ├── decorators/
│   │   └── security.py            # @signature_required (HMAC-SHA256 validation)
│   ├── utils/
│   │   └── whatsapp_utils.py      # Triage state machine, Gemini logic & Meta API client
│   └── views.py                   # Blueprint for GET & POST /webhook
├── test_gemini.py                 # Standalone Gemini SDK verification script
├── .env.example                   # Environment configuration template
├── requirements.txt               # Application dependencies
└── run.py                         # Application entrypoint (starts threaded Flask app)

```

---

## Configuration & Environment Variables

All settings are loaded into the Flask context via `app/config.py`. Create a `.env` file in the root directory:

```bash
cp .env.example .env

```

Configure the following variables:

```env
# Meta WhatsApp Cloud API
ACCESS_TOKEN=your_meta_system_user_token
APP_ID=your_meta_app_id
APP_SECRET=your_meta_app_secret
VERSION=v18.0
PHONE_NUMBER_ID=your_whatsapp_phone_number_id
YOUR_PHONE_NUMBER=your_test_phone_number
RECIPIENT_WAID=your_default_recipient_phone_number

# Webhook Handshake Verification
VERIFY_TOKEN=your_custom_verification_token_string

# Google Gemini API
GEMINI_API_KEY=your_gemini_api_key

# Incident Alerting & Multi-Responder Topology
# Admin receives complete dossier with victim PII and Google Maps link
ADMIN_NUMBER=919876543210

# Responders receive sanitized alerts (comma-separated list)
RESPONDER_NUMBERS=919876543211,919876543212,919876543213

# Central Monitoring Dashboard Integration (Optional)
DASHBOARD_API_URL=[https://your-incident-dashboard.com/api/v1/incidents](https://your-incident-dashboard.com/api/v1/incidents)

```

---

## Installation & Setup

### 1. Clone the Repository

```bash
git clone [https://github.com/your-username/your-repo-name.git](https://github.com/your-username/your-repo-name.git)
cd your-repo-name

```

### 2. Set Up Virtual Environment

```bash
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

```

### 3. Install Dependencies

```bash
pip install -r requirements.txt

```

*(Ensure `requirements.txt` includes: `flask`, `requests`, `python-dotenv`, `google-generativeai`)*

### 4. Verify Gemini API Connection (Optional)

Run the standalone diagnostic script to ensure your Gemini API key is valid:

```bash
python test_gemini.py

```

### 5. Launch the Server

```bash
python run.py

```

The server will start listening on `http://0.0.0.0:8000`.

---

## Meta Webhook Configuration

Meta requires an active public HTTPS endpoint to deliver webhook notifications.

### 1. Launch ngrok Tunnel

```bash
ngrok http 8000 --domain your-domain.ngrok-free.app

```

### 2. Configure Meta Developer Portal

1. Go to the [Meta App Dashboard](https://developers.facebook.com/apps/) > **WhatsApp** > **Configuration**.
2. Under **Webhook**, click **Edit**:
* **Callback URL**: `https://your-domain.ngrok-free.app/webhook`
* **Verify Token**: Must match the `VERIFY_TOKEN` in your `.env`.


3. Click **Verify and Save**. Your terminal will log:
```text
INFO:root:WEBHOOK_VERIFIED

```


4. Click **Manage** under Webhook Fields and check the box for **`messages`**.

---

## Triage State Machine Reference

| Stage | Trigger / Input | State | Action Taken |
| --- | --- | --- | --- |
| **Idle** | Standard text or question | `None` | Evaluated through standard Gemini 2.5 Flash prompt. |
| **Trigger** | Starts with `"HELP"` | `awaiting_location` | Initializes session state; prompts user for live GPS pin or address. |
| **Location** | Sends GPS location pin or text address | `awaiting_injury` | Extracts coordinates or address text; prompts for injury status. |
| **Injuries** | `"Yes"`, `"Bleeding"`, `"No"`, etc. | `awaiting_people_count` | Normalizes injury flag; prompts for number of affected persons. |
| **Headcount** | Number (e.g., `"4"`, `"just me"`) | `location_received` | Fires admin dossier, responder broadcast, and dashboard JSON sync. |
| **Active De-escalation** | Follow-up text | `location_received` | Runs calm, 1-2 sentence crisis Gemini prompt with chat history. |

### Dispatch Formats

#### 1. Administrator Dossier (`ADMIN_NUMBER`)

```text
NEW EMERGENCY
From: John Doe (+919876543210)
Location: [https://maps.google.com/?q=17.4374,78.3842](https://maps.google.com/?q=17.4374,78.3842)
Injured: yes
People: 4

```

#### 2. Responder Broadcast (`RESPONDER_NUMBERS`)

```text
New emergency reported. Check the admin/dashboard for full details.

```

#### 3. Dashboard Webhook Payload (`DASHBOARD_API_URL`)

```json
{
  "name": "John Doe",
  "phone": "919876543210",
  "latitude": 17.4374,
  "longitude": 78.3842,
  "address_text": null,
  "injured": "yes",
  "people_count": 4,
  "timestamp": "2026-09-09T22:30:00.000000"
}

```

---

## Security Implementation

### Webhook Verification Handshake

Meta performs an initial `GET` request containing `hub.mode`, `hub.verify_token`, and `hub.challenge`. The server validates `hub.verify_token` against `current_app.config["VERIFY_TOKEN"]` and returns `hub.challenge` with status **`200 OK`**.

### Request Signature Verification

Incoming `POST` webhook requests are intercepted by `@signature_required` in `app/decorators/security.py`:

* Extracts the signature hash from the `X-Hub-Signature-256` header (stripping the `sha256=` prefix).
* Generates an expected HMAC-SHA256 signature using the application's `APP_SECRET` and the raw request body.
* Uses constant-time string comparison (`hmac.compare_digest`) to prevent timing attacks. Requests with invalid signatures are rejected with **`403 Forbidden`**.

---

## Production Deployment Checklist

* [ ] **WSGI Server**: Run behind Gunicorn (`gunicorn -w 4 -b 0.0.0.0:8000 run:app`).
* [ ] **State Persistence**: For multi-worker deployments, migrate `active_emergencies = {}` and `_seen_message_ids` from in-memory dictionaries to **Redis** to ensure state persistence across worker threads.
* [ ] **Meta System User Token**: Ensure your access token is generated from a System User with permanent/60-day validity to avoid mid-operation auth failures.
* [ ] **Reverse Proxy**: Configure Nginx with SSL termination via Let's Encrypt for direct production domain handling.

```

```
