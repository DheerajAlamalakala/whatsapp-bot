# Raksha (रक्षा) — AI WhatsApp Safety & Emergency Response Bot

**Raksha** is an intelligent, real-time personal safety and emergency support assistant built on the **Meta WhatsApp Cloud API**, **Python**, **Flask**, and **OpenAI**. It serves as an accessible lifeline directly within WhatsApp—enabling users to trigger emergency alerts, share real-time location details, and receive safety assistance through conversational AI.

---

## Table of Contents

- [About Raksha](#about-raksha)
- [Key Features](#key-features)
- [Architecture & Tech Stack](#architecture--tech-stack)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Setup & Installation](#setup--installation)
- [Step-by-Step Configuration](#step-by-step-configuration)
  - [1. Meta WhatsApp Cloud API Setup](#1-meta-whatsapp-cloud-api-setup)
  - [2. Environment Variables](#2-environment-variables)
  - [3. Run Application Locally](#3-run-application-locally)
  - [4. Expose Webhook via ngrok](#4-expose-webhook-via-ngrok)
  - [5. Verify & Subscribe to Webhook](#5-verify--subscribe-to-webhook)
- [Security & Webhook Validation](#security--webhook-validation)
- [AI Integration](#ai-integration)
- [Production Deployment & Migration](#production-deployment--migration)
- [Contributing & License](#contributing--license)

---

## About Raksha

The name **Raksha** translates to *protection*. During critical moments, navigating complex mobile applications can be difficult or unfeasible. WhatsApp offers an immediate, low-bandwidth channel that most users already rely on daily. Raksha leverages WhatsApp to:
- Act on distress keywords (e.g., `EMERGENCY`, `HELP`, `SOS`).
- Provide AI-guided safety instructions, de-escalation tips, and emergency helpline details.
- Securely receive and forward live coordinates or status updates to emergency contacts.

---

## Key Features

- **WhatsApp Cloud API Integration**: Direct interaction via Meta's official Graph API.
- **Flask Webhook Backend**: Lightweight, modular webhook event receiver.
- **HMAC-SHA256 Payload Validation**: Enforces cryptographic request signing (`X-Hub-Signature-256`) to reject forged incoming payloads.
- **Intelligent Safety Responses**: Powered by OpenAI to process conversational text, deliver step-by-step assistance, or route emergency queries.
- **Extensible Emergency Routing**: Modular service structure allowing integration with SMS gateways, Twilio, or emergency dispatch services.

---

## Architecture & Tech Stack

- **Language**: Python 3.10+
- **Framework**: Flask
- **External APIs**: Meta Graph API (WhatsApp Cloud API v18.0+), OpenAI Assistants / Chat API
- **Tunneling / Development**: ngrok
- **Security**: Cryptographic verification via HMAC SHA-256

---

## Prerequisites

1. **Meta Developer Account**: Register at [developers.facebook.com](https://developers.facebook.com/).
2. **Meta Business App**: Create a business app and add the **WhatsApp** product.
3. **OpenAI API Key**: Create an API key at [platform.openai.com](https://platform.openai.com/).
4. **ngrok Account**: Required for local webhook development with static domains.
5. **Python 3.10+** and `pip` installed.

---

## Project Structure

```text
raksha/
├── app/
│   ├── __init__.py
│   ├── decorators/
│   │   └── security.py        # Webhook signature & token verification
│   ├── services/
│   │   └── openai_service.py  # AI query handler and prompt logic
│   ├── utils/
│   │   └── whatsapp_utils.py  # Message formatting, payload processing, sender
│   └── views.py               # Flask webhook endpoints (GET & POST)
├── .env.example
├── requirements.txt
├── run.py                     # Entry point
└── README.md
