# Telegram → ClickUp Task Automation — Setup Guide

## Overview

This n8n workflow listens to your Telegram project group chats. When a message assigns a task to a team member by Persian name, it automatically:
1. Analyzes the message with Claude AI
2. Finds the matching ClickUp user (fuzzy name matching)
3. Identifies the current sprint in the correct project folder
4. Creates the ClickUp task with full details
5. Replies in the Telegram group with a confirmation in Persian

---

## Deliverable 2 — Credentials You Need

| Credential | Where to get it | n8n credential type |
|---|---|---|
| **Telegram Bot Token** | Create a bot via [@BotFather](https://t.me/BotFather), use the token it gives you | `Telegram API` |
| **ClickUp Personal API Token** | ClickUp → Profile → Apps → Generate Personal API Token | `HTTP Header Auth` (header name: `Authorization`, value: your token) |
| **Anthropic API Key** | [console.anthropic.com](https://console.anthropic.com) → API Keys | `HTTP Header Auth` (header name: `x-api-key`, value: your key) |

### Setting up credentials in n8n

1. Open n8n → **Credentials** → **New Credential**
2. For ClickUp: choose **HTTP Header Auth**, set Name=`Authorization`, Value=`pk_YOUR_TOKEN`
3. For Anthropic: choose **HTTP Header Auth**, set Name=`x-api-key`, Value=`sk-ant-YOUR_KEY`
4. For Telegram: choose **Telegram API**, paste your bot token
5. After import, open each node and re-select the matching credential from the dropdown

---

## Deliverable 3 — Configuration Table to Fill In

Edit the **`Lookup Project`** Code node and replace the placeholder values:

```
TELEGRAM_CHAT_ID  →  CLICKUP_FOLDER_ID  →  PROJECT_NAME
```

| Telegram Group | Chat ID | ClickUp Folder ID | Notes |
|---|---|---|---|
| Skylife | `-100XXXXXXXXX` | `FOLDER_ID_SKYLIFE` | Replace both values |
| Insight | `-100XXXXXXXXX` | `FOLDER_ID_INSIGHT` | |
| Homeland | `-100XXXXXXXXX` | `FOLDER_ID_HOMELAND` | |
| Skyparadise | `-100XXXXXXXXX` | `FOLDER_ID_SKYPARADISE` | |
| LiveElite | `-100XXXXXXXXX` | `FOLDER_ID_LIVEELITE` | |
| The Hub | `-100XXXXXXXXX` | `FOLDER_ID_THEHUB` | |
| Ghelichkhani | `-100XXXXXXXXX` | `FOLDER_ID_GHELICHKHANI` | |
| Shafiee | `-100XXXXXXXXX` | `FOLDER_ID_SHAFIEE` | |
| Qattali | `-100XXXXXXXXX` | `FOLDER_ID_QATTALI` | |
| Dorna | `-100XXXXXXXXX` | `FOLDER_ID_DORNA` | |
| Arta Group | `-100XXXXXXXXX` | `FOLDER_ID_ARTAROUP` | |
| Caribbeanparadise | `-100XXXXXXXXX` | `FOLDER_ID_CARIBBEANPARADISE` | |

You also need to set the n8n **workflow variable** `CLICKUP_TEAM_ID`:  
Go to **Settings → Variables** in n8n and add `CLICKUP_TEAM_ID = your_team_id`.

---

## Deliverable 4 — The Exact AI Prompt

The **System Prompt** sent to Claude (inside the `Analyze Message (Claude)` node):

```
You are a bilingual assistant (Persian/English) that analyzes Persian Telegram messages
from project management groups. Your only job is to extract task assignment information
and return strictly valid JSON with no extra text, no markdown, no code fences.

Output EXACTLY this JSON schema:
{
  "is_task": boolean,
  "assignee_persian": string | null,
  "assignee_english": string | null,
  "title": string | null,
  "description": string | null,
  "priority": "urgent" | "high" | "normal" | "low",
  "due_date": "YYYY-MM-DD" | null
}

Rules:
- Set is_task=true only if the message assigns a specific action to a named person
- assignee_persian: Persian/Farsi name as written in message
- assignee_english: Romanized transliteration
  (e.g. نوید→Navid, آرش→Arash, سارا→Sara, علی→Ali, محمد→Mohammad,
   رضا→Reza, فاطمه→Fatemeh, زهرا→Zahra, مهدی→Mehdi, حسین→Hossein,
   امیر→Amir, شهاب→Shahab, نیلوفر→Niloofar, پریسا→Parisa,
   سینا→Sina, کامران→Kamran, بهزاد→Behzad)
- title: concise English imperative sentence (max 80 chars)
- description: clear English summary of what needs to be done
- priority: infer from urgency words
  (فوری/ASAP=urgent, مهم/important=high, عادی=normal, کم‌اهمیت=low).
  Default: normal
- due_date: if mentioned extract as YYYY-MM-DD, null otherwise
- If is_task=false, all other fields should be null except priority="normal"
```

The **User Message** sent per execution:

```
Analyze this Telegram message from the '[projectName]' project group:

Message: [messageText]
Sender: [senderName]
Date: [YYYY-MM-DD]
```

### Example input/output pairs

**Input:**
```
نوید لطفاً باگ لاگین رو تا فردا درست کن فوریه
```
**Output:**
```json
{
  "is_task": true,
  "assignee_persian": "نوید",
  "assignee_english": "Navid",
  "title": "Fix login bug",
  "description": "Navid needs to fix the login bug by tomorrow. Marked as urgent.",
  "priority": "urgent",
  "due_date": "2026-05-04"
}
```

**Input:**
```
آرش داشبورد آنالیتیکس رو تا آخر هفته آپدیت کن
```
**Output:**
```json
{
  "is_task": true,
  "assignee_persian": "آرش",
  "assignee_english": "Arash",
  "title": "Update analytics dashboard",
  "description": "Arash should update the analytics dashboard by end of the week.",
  "priority": "normal",
  "due_date": "2026-05-09"
}
```

**Input:**
```
جلسه فردا ساعت ۱۰ صبح یادتون باشه
```
**Output:**
```json
{
  "is_task": false,
  "assignee_persian": null,
  "assignee_english": null,
  "title": null,
  "description": null,
  "priority": "normal",
  "due_date": null
}
```

---

## Deliverable 5 — Fuzzy Matching Code (standalone reference)

This lives in the **`Match ClickUp User`** Code node. Paste here for reference:

```javascript
function levenshtein(a, b) {
  const m = a.length, n = b.length;
  const dp = [];
  for (let i = 0; i <= m; i++) {
    dp[i] = [i];
    for (let j = 1; j <= n; j++) dp[i][j] = i === 0 ? j : 0;
  }
  for (let j = 1; j <= n; j++) dp[0][j] = j;
  for (let i = 1; i <= m; i++) {
    for (let j = 1; j <= n; j++) {
      dp[i][j] = a[i-1] === b[j-1]
        ? dp[i-1][j-1]
        : 1 + Math.min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1]);
    }
  }
  return dp[m][n];
}

function similarity(a, b) {
  a = a.toLowerCase().trim();
  b = b.toLowerCase().trim();
  const maxLen = Math.max(a.length, b.length);
  return maxLen === 0 ? 1.0 : 1 - levenshtein(a, b) / maxLen;
}

// For each ClickUp member, compare against: username, email prefix, name parts
// Pick the highest scoring field. Threshold = 0.70 (70%)
```

**Why 0.70?** It allows for:
- Minor transliteration variations: `Navid` vs `Naveed` → ~0.83
- Short names with one typo: `Ali` vs `Aly` → 0.67 (would miss — intentional, too ambiguous)
- Email prefix matching: `arash.tehrani@co.com` → prefix `arash.tehrani` split to `arash` matches `Arash` at 1.0

Adjust the threshold in the Code node if you get too many false negatives.

---

## Deliverable 6 — Finding Your ClickUp IDs via API

### Step 1 — Get your Team ID

```bash
curl -H "Authorization: pk_YOUR_TOKEN" \
     https://api.clickup.com/api/v2/team
```

Response: look for `"teams": [{ "id": "YOUR_TEAM_ID", "name": "Insight's Workspace" }]`

### Step 2 — Get Space IDs (find "IPBs")

```bash
curl -H "Authorization: pk_YOUR_TOKEN" \
     https://api.clickup.com/api/v2/team/YOUR_TEAM_ID/space?archived=false
```

Look for `"name": "IPBs"` and note its `"id"`.

### Step 3 — Get Folder IDs for each project

```bash
curl -H "Authorization: pk_YOUR_TOKEN" \
     https://api.clickup.com/api/v2/space/IPBS_SPACE_ID/folder?archived=false
```

Response lists all folders. Match by `"name"` to your projects:
```json
{
  "folders": [
    { "id": "12345678", "name": "Skylife" },
    { "id": "23456789", "name": "Insight" },
    ...
  ]
}
```

Copy each folder's `"id"` into the mapping table in Step 3 above.

### Step 4 — Find Telegram Chat IDs

Add your bot to each group, then either:
- Send a message in the group and check: `https://api.telegram.org/botYOUR_TOKEN/getUpdates`
- Or use [@getidsbot](https://t.me/getidsbot) — forward a message from the group to it

Chat IDs for groups are negative numbers like `-1001234567890`.

### Step 5 — Set the n8n variable

In n8n: **Settings → Variables → Add Variable**
- Name: `CLICKUP_TEAM_ID`
- Value: your team ID from Step 1

---

## Importing the Workflow

1. In n8n, go to **Workflows → Import from File**
2. Select `workflow.json`
3. Open each HTTP Request node and re-assign the credentials
4. Edit the **`Lookup Project`** Code node — fill in your chat IDs and folder IDs
5. Set the `CLICKUP_TEAM_ID` variable (Settings → Variables)
6. **Activate** the workflow (toggle top-right)
7. Make sure your Telegram bot has been added as an admin to each project group  
   (it needs permission to read messages and send replies)

---

## Workflow Diagram

```
Telegram Trigger
      │
Extract Message Data (Set)
      │
Lookup Project (Code)  ──[not found]──→ No Action (Unknown Group)
      │ found
Project Found? (IF)
      │ true
Analyze Message / Claude (HTTP)
      │
Parse Claude Response (Code)
      │
Is Task? (IF) ──[false]──→ No Action (Not a Task)
      │ true
Get ClickUp Members (HTTP)
      │
Match ClickUp User (Code)
      │
User Found? (IF) ──[false]──→ Send User Not Found Error (Telegram)
      │ true
Get Sprint Lists (HTTP)
      │
Find Current Sprint (Code)
      │
Build Task Body (Set)
      │
Create ClickUp Task (HTTP)  ──[error]──→ Task Creation Error Handler → Send Error (Telegram)
      │ success
Send Success Reply (Telegram)
```

---

## Notes & Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| Bot doesn't receive messages | Bot not added to group / not admin | Add bot to group, grant admin rights |
| "Project not found" in logs | Chat ID mismatch | Re-check chat ID (use getUpdates API) |
| User never matches | Name transliteration differs from ClickUp username | Lower threshold to 0.60, or add the user's exact ClickUp username to the mapping |
| Sprint not found | Sprint names don't match pattern | Check exact sprint name format in ClickUp; update regex in `Find Current Sprint` if needed |
| Claude returns non-JSON | Rare model behavior | The `Parse Claude Response` node will throw; workflow stops. Retry usually succeeds |
| ClickUp 401 error | Token expired or wrong format | Regenerate token; ensure header value is `pk_xxxxx` (no "Bearer" prefix for ClickUp) |
