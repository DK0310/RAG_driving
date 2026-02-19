# 🚗 RADI — Vietnam Driving Theory RAG Chatbot

A retrieval-augmented generation (RAG) chatbot built on **n8n**, **Supabase (PostgreSQL)**, and **Google Gemini** to help Vietnamese learners study and practice driving theory (lý thuyết lái xe GPLX).

Users interact with RADI directly through **Facebook Messenger**.

---

## ✨ Features

- 🎯 **Question Practice** — Pulls from a full 600-question bank organized by chapter
- 📝 **Exam Simulation** — Timed mock exams for licenses A1/A, B1, B, C/D/E/F with auto-grading and "liệt" (disqualifying) question detection
- 🖼️ **Image Support** — Detects when questions have diagrams or road sign images and sends them inline
- ✅ **Answer Checking** — Validates user answers against the `keys` database with explanations
- 💬 **Conversational Memory** — PostgreSQL-backed chat history per user session
- 🔄 **Data Refresh** — Admins can trigger live data updates from Google Drive/Docs
- 👋 **New User Onboarding** — Auto-detects first-time users and sends a welcome message
- 🛡️ **Duplicate Message Guard** — Deduplicates incoming Messenger webhooks via `message_id`

---

## 🏗️ Architecture

```
Facebook Messenger
       │
       ▼
  n8n Webhook  ──► Verify (GET hub.challenge)
       │
       ▼
  Message Guard (dedup by message_id)
       │
  ┌────┴────────────────────┐
  │                         │
New User?             Existing User
  │                         │
INSERT users          Check history
  │                         │
Welcome msg      ┌──────────┴──────────┐
              "cập nhật dữ liệu"?   Normal message
                  │                    │
              AI Agent3           AI Agent2
          (Data Refresh)      (Main RAG Agent)
                  │                    │
              Gemini Flash        Gemini Flash
                                       │
                              ┌────────┴────────┐
                          Text only        Has image URL
                              │                 │
                         Send text        Download from
                                          Google Drive
                                               │
                                          Send image
```

---

## 🗄️ Database Schema (Supabase PostgreSQL)

| Table | Purpose |
|---|---|
| `users` | Registered user IDs and join timestamps |
| `facebook_chat_histories` | Incoming message log (dedup by `message_id`) |
| `n8n_chat_histories` | LangChain conversation memory per session |
| `questions` | Full question bank (`question_id`, `content`, `key_id`, `bool_liet`, `chapter_id`) |
| `chapters` | Chapter metadata |
| `keys` | Answer keys |
| `user_questions` | Questions that have been served to a user (for context) |

---

## 🧠 AI Agents

### AI Agent2 — Main RAG Agent
The core conversational agent. Uses **Gemini 2.5 Flash** and has access to:
- `questions` — Full question database
- `keys` — Answer key database
- `chapters` — Chapter metadata
- `user_questions` — Previously served questions (for answer checking context)
- `Interaction Rules` (Google Doc) — Tone and conversation guidelines
- `RADI Knowledge` (Google Doc) — Driving theory domain knowledge
- `Image data` (Google Sheet) — Question-to-image URL mapping
- PostgreSQL Chat Memory — Persistent session memory

### AI Agent3 — Data Refresh Agent
Triggered by the phrase **"cập nhật dữ liệu"**. Calls the `read_drive_folder` sub-workflow to pull updated files from Google Drive and confirms success to the user.

---

## 🔀 Workflow Flow

1. **Webhook** receives Messenger POST → responds to GET for webhook verification
2. **If1** — checks if message has text (ignores reactions, stickers, etc.)
3. **Postgres** — checks if `user_id` exists in `users` table
4. **If3** — routes new vs. returning users
5. **Postgres2** — checks `facebook_chat_histories` for duplicate `message_id`
6. **If4** — skips processing if already handled
7. **Insert to history messages** — logs the new message
8. **If** — checks for "cập nhật dữ liệu" keyword → routes to Agent3 or Agent2
9. **AI Agent2** — generates a response
10. **IfObject / IfImage** — detects if response contains an image URL
11. **Image path** — downloads from Google Drive, sends via Messenger attachment API
12. **Text path** — sends plain text response via Messenger Graph API
13. **Lưu questions** — saves the AI's question output to `user_questions` for answer-checking context

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Workflow Automation | [n8n](https://n8n.io) |
| LLM | Google Gemini 2.5 Flash |
| Database | Supabase (PostgreSQL) |
| Chat Memory | n8n LangChain PostgreSQL Memory |
| Messaging Platform | Facebook Messenger (Graph API v23.0) |
| Knowledge Base | Google Docs + Google Sheets |
| File Storage | Google Drive |

---

## ⚙️ Setup

### Prerequisites
- n8n instance (self-hosted or cloud)
- Supabase project with the tables listed above
- Facebook App with Messenger permissions and a verified webhook
- Google Cloud project with OAuth2 credentials for Docs, Sheets, and Drive
- Google Gemini API key

### Environment / Credentials to Configure in n8n

| Credential | Used By |
|---|---|
| `Postgres account` | All database nodes |
| `Google Docs account` (OAuth2) | Interaction Rules, RADI Knowledge, Update data tools |
| `Google Sheets account` (OAuth2) | Image data tool |
| `Google Drive account` (OAuth2) | Image download nodes |
| `Google Gemini (PaLM) API` | All LLM nodes |
| Facebook Page Access Token | All `HTTP Request` nodes sending to Messenger |

### Facebook Webhook
- **URL:** `<your-n8n-base-url>/webhook/e593c452-d1e1-47ab-b982-649138fac915`
- **Verify Token:** Set in your Facebook App and matched in the webhook `hub.challenge` response node
- **Subscribed Fields:** `messages`

---

## 💬 User Commands

| Input | Behavior |
|---|---|
| `Bắt đầu` | Start and choose license type |
| `Luyện chương 1` | Practice questions from Chapter 1 |
| `Thi thử A1` | Start A1/A exam simulation (25 questions) |
| `Thi thử B` | Start B exam simulation (35 questions) |
| `Câu 37` | Get question number 37 |
| `Ảnh 37` | Get image for question 37 |
| `1`, `2`, `3`, `4` | Answer a multiple-choice question |
| `cập nhật dữ liệu` | (Admin) Trigger data refresh from Google Drive |

---

## 📁 Project Structure

```
RAGmess (n8n Workflow)
├── Webhook Entry (Messenger)
├── User Management
│   ├── Postgres (user exists check)
│   ├── Postgres1 (insert new user)
│   └── Code (welcome message)
├── Message Deduplication
│   ├── Postgres2 (check message_id)
│   └── Insert to history messages
├── Routing
│   ├── If (data refresh keyword)
│   └── AI Agent3 (refresh flow)
├── Main Agent (AI Agent2)
│   ├── Tool: questions
│   ├── Tool: keys
│   ├── Tool: chapters
│   ├── Tool: user_questions
│   ├── Tool: Interaction Rules (Google Doc)
│   ├── Tool: RADI Knowledge (Google Doc)
│   └── Tool: Image data (Google Sheet)
├── Image Handling
│   ├── IfObject / IfImage / IfImage2
│   ├── Google Drive download
│   └── Messenger attachment send
└── Response Delivery
    ├── Loop Over Items (batch send)
    └── Lưu questions (save context)
```

---

## 📌 Notes

- The workflow uses **multiple Facebook Page Access Tokens** across different HTTP nodes — make sure all tokens are kept up to date as they expire.
- The `user_questions` table acts as short-term working memory so the AI can check the user's answer against the question it just served.
- `bool_liet = TRUE` in the `questions` table flags disqualifying questions — answering these incorrectly during an exam simulation results in immediate failure regardless of total score.
- Image URLs are stored in Google Drive and fetched/downloaded at response time, then forwarded to Messenger as binary attachments.

---

## 📄 License

Personal project — MIT license.
