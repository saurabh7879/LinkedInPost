# LinkedIn Post Generator — Setup

## Overview

Every day at 9 AM IST (or when the webhook is called), the workflow picks a topic from `topics.csv`, has OpenAI write a LinkedIn post in a fixed format, and publishes it to your LinkedIn profile.

**Stack:** n8n · OpenAI `gpt-4o-mini` · LinkedIn Posts API (`/rest/posts`)

---

## Workflow Architecture

```
Daily 9 AM IST ─┐
                ├─► Fetch CSV ─► Parse CSV ─► Topic Exists? ──NO──► No Topics Response
POST webhook ───┘   (GitHub raw)  (topic of     │ YES
                                  the day)      ▼
                                           Build Prompt
                                                ▼
                                    Generate Post (OpenAI)
                                                ▼
                                       Format for LinkedIn   escapes reserved characters,
                                                ▼            turns #tags into hashtags
                                       Publish to LinkedIn   POST /rest/posts
                                                ▼
                                         Success Response    post ID (x-restli-id) + text
```

Topics rotate one per day: `dayIndex % number_of_rows` (IST date). The CSV is read from
`https://raw.githubusercontent.com/saurabh7879/LinkedInPost/develop/topics.csv`. Edit
the **Fetch CSV** node URL to use another file (it must be publicly readable).

---

## Step 1 — LinkedIn access token

1. Go to the [LinkedIn Developer Portal](https://www.linkedin.com/developers/apps) and open your app.
2. **Products** tab: add **Share on LinkedIn** (gives `w_member_social`) and
   **Sign In with LinkedIn using OpenID Connect** (gives `openid profile`).
3. **Auth** tab → **OAuth 2.0 tools** → **Create token**. Select `w_member_social openid profile`.
4. Copy the token. **It expires after 60 days** — repeat these steps and update the n8n credential when it does.

### Check your person URN
```bash
curl -H "Authorization: Bearer YOUR_TOKEN" https://api.linkedin.com/v2/userinfo
```
Your URN is `urn:li:person:<sub>`. It must match `AUTHOR_URN` at the top of the
**Format for LinkedIn** node (currently `urn:li:person:1H9p-wtG1P`).

---

## Step 2 — Create two credentials in n8n

n8n → **Credentials** → **Add credential** → **Header Auth**, twice:

| Credential name | Header name | Header value |
|---|---|---|
| `OpenAI Bearer` | `Authorization` | `Bearer sk-...your OpenAI key` |
| `LinkedIn Bearer` | `Authorization` | `Bearer AQ...your LinkedIn token` |

Keys live only in n8n, never in the workflow JSON. A local `.env` (ignored by git) can hold
copies for reference — never commit it.

---

## Step 3 — Import & activate

1. n8n → **+ New Workflow** → **⋯** → **Import from File** → `linkedin_post_workflow.json`.
2. Open **Generate Post** and select the `OpenAI Bearer` credential; open **Publish to LinkedIn**
   and select `LinkedIn Bearer` (n8n may ask you to re-pick them after import).
3. Save and toggle **Active**.

If you already have an older version imported, delete it (or deactivate it) so it doesn't post twice.

---

## Step 4 — Test

Run the workflow manually in the editor, or call the webhook:

```bash
curl -X POST https://n8n.framegenai.cloud/webhook/linkedin-post
```

The topic comes from the CSV; the request body is not used.

**Success response:**
```json
{
  "status": "success",
  "topic": "Designing scalable APIs in ASP.NET Core with caching and Redis",
  "linkedInPostId": "urn:li:share:123456789",
  "postContent": "🚀 ..."
}
```

---

## Troubleshooting

| Error from Publish to LinkedIn | Fix |
|---|---|
| `401 EXPIRED_ACCESS_TOKEN` | Token is over 60 days old — create a new one (Step 1) and update `LinkedIn Bearer`. |
| `401 EMPTY_ACCESS_TOKEN` / `INVALID_ACCESS_TOKEN` | Credential not selected, or header value missing the `Bearer ` prefix. |
| `403 ACCESS_DENIED` | Token lacks `w_member_social`, or `AUTHOR_URN` is not the token owner's URN. |
| `426` / `NONEXISTENT_VERSION` | `LinkedIn-Version` header is too old. Set it to a current month (`YYYYMM`) — LinkedIn supports each version for about a year. |
| `400 FIELD_LENGTH_TOO_LONG` | Post over 3000 characters (the formatter trims to 2900, so this is rare). |
| Post text cut off | A reserved character wasn't escaped — check the **Format for LinkedIn** output. |

Docs: [Posts API](https://learn.microsoft.com/linkedin/marketing/community-management/shares/posts-api) ·
[little text format](https://learn.microsoft.com/linkedin/marketing/community-management/shares/little-text-format) ·
[Share on LinkedIn](https://learn.microsoft.com/linkedin/consumer/integrations/self-serve/share-on-linkedin)
