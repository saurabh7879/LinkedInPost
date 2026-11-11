# LinkedIn Post Generator — Setup

## Overview

Accepts a topic via webhook, generates a professional LinkedIn post using Gemini AI, and publishes it directly to your LinkedIn profile.

**Stack:** n8n · Gemini 2.5 Flash · LinkedIn UGC Posts API

---

## Workflow Architecture

```
POST /webhook/linkedin-post
        │
        ▼
┌─────────────────┐
│   Set Inputs    │  topic, tone, audience (with defaults)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Topic Provided? │──── NO ────► Error response
└────────┬────────┘
         │ YES
         ▼
┌─────────────────┐
│  Generate Post  │  Gemini: writes LinkedIn-style post with hashtags
│  (Gemini AI)    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Extract Post   │  Pulls clean text from Gemini response
└────────┬────────┘
         │
         ▼
┌─────────────────────┐
│ Publish to LinkedIn │  LinkedIn UGC Posts API
└────────┬────────────┘
         │
         ▼
┌─────────────────┐
│ Success Response│  Returns post ID + content
└─────────────────┘
```

---

## Step 1 — Get LinkedIn Credentials

### Access Token
1. Go to [LinkedIn Developer Portal](https://www.linkedin.com/developers/)
2. Create an app → request `w_member_social` permission
3. Use OAuth 2.0 to get an access token
4. Token expires in 60 days — refresh as needed

### Person URN
Run this after getting your token:
```bash
curl -H "Authorization: Bearer YOUR_TOKEN" \
  "https://api.linkedin.com/v2/me"
```
Copy the `id` field — your URN is `urn:li:person:{id}`

---

## Step 2 — Add Environment Variables in n8n

For convenience you can create a `.env` file at the project root (do NOT commit this file). Example `.env` content:

```
GEMINI_API_KEY=your_gemini_key
LINKEDIN_ACCESS_TOKEN=your_linkedin_token
LINKEDIN_PERSON_URN=urn:li:person:your_id
```

Alternatively, set these variables directly in n8n credentials or your deployment environment. If you create a `.env` file, this project already ignores it via `.gitignore`.

---

## Step 3 — Import & Activate

1. n8n dashboard → **+ New Workflow** → Import `linkedin_post_workflow.json`
2. Toggle **Activate** to green

---

## Step 4 — Test

```bash
curl -X POST https://n8n.framegenai.cloud/webhook/linkedin-post \
  -H "Content-Type: application/json" \
  -d '{"topic": "AI trends in 2025", "tone": "professional", "audience": "tech entrepreneurs"}'
```

**Request body fields:**

| Field | Required | Default | Example |
|---|---|---|---|
| `topic` | YES | — | `"The future of remote work"` |
| `tone` | no | `professional` | `"inspirational"`, `"casual"`, `"bold"` |
| `audience` | no | `professionals and entrepreneurs` | `"software developers"` |

**Success response:**
```json
{
  "status": "success",
  "message": "Post published to LinkedIn",
  "linkedInPostId": "urn:li:share:123456789",
  "postContent": "AI is not replacing jobs..."
}
```

---

## Tone Options

| Tone | Use for |
|---|---|
| `professional` | Industry insights, career tips |
| `inspirational` | Motivational stories, personal wins |
| `casual` | Relatable everyday observations |
| `bold` | Controversial takes, hot opinions |
| `educational` | How-tos, step-by-step guides |
| `storytelling` | Personal experiences, case studies |
