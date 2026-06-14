# ContextMail Architecture 🏗️

A technical deep-dive into how ContextMail works under the hood.

---

## System Overview

```
┌──────────────────────────────────────────────────────────────┐
│                    External Services                         │
│  Gmail │ OpenAI │ Google Sheets │ Telegram │ n8n              │
└─────────────────┬────────────────────────────────────────────┘
                  │
        ┌─────────▼──────────────┐
        │   ContextMail Workflow  │
        │   (n8n Orchestration)   │
        └─────────┬──────────────┘
                  │
        ┌─────────┴─────────────────────┐
        │                               │
        ▼ Email Processing              ▼ User Query Handler
   (Incoming Emails)              (Telegram Interactions)
```

---

## Core Components

### 1. **Email Processing Pipeline**

#### A. Gmail Trigger
- **Type**: `n8n-nodes-base.gmailTrigger`
- **Version**: 1.4
- **Function**: Monitors Gmail inbox for new emails
- **Configuration**:
  - Poll interval: Every 1 minute
  - Credentials: Gmail OAuth2 API
  - Output: Email object with metadata

**Email Data Structure:**
```javascript
{
  "From": "sender@example.com",
  "Subject": "Meeting Tomorrow",
  "snippet": "Hi, let's meet about the project...",
  "threadId": "abc123def456",
  "date": "2026-06-14T10:30:00Z"
}
```

#### B. Summary Agent (OpenAI)
- **Type**: `@n8n/n8n-nodes-langchain.openAi`
- **Version**: 2.3
- **Model**: `gpt-5-mini`
- **Function**: Generates structured email summaries
- **Process**:
  1. Takes email data from Gmail Trigger
  2. Passes to OpenAI with detailed prompt
  3. Returns 4-section formatted summary

**Prompt Template:**
```
Summarize email with 4 sections:
🎯 The Pitch (who & what in 1-2 sentences)
📋 The Details (main proposal in 2-3 sentences)
⏰ The Timeline (deadlines or "no timeline")
🚨 Priority (High/Medium/Low + reason)
```

**Output Example:**
```
🎯 The Pitch
Hey! Sarah from Sales just emailed about the Q3 proposal review.

📋 The Details
She wants feedback on the deck by Friday. Seems straightforward.

⏰ The Timeline
Friday 5 PM is the deadline.

🚨 Priority
High – Deadline this week, client-facing work.
```

#### C. Google Sheets Node (Append)
- **Type**: `n8n-nodes-base.googleSheets`
- **Version**: 4.7
- **Function**: Stores email + summary in database
- **Columns Appended**:
  ```
  Sender_Email: {{ Gmail Trigger.From }}
  Subject: {{ Gmail Trigger.Subject }}
  Full_Email_Content: {{ Gmail Trigger.snippet }}
  AI_Summary: {{ Summary Agent.output[0].content[0].text }}
  Received_Time: {{ $now }}
  Thread_ID: {{ Gmail Trigger.threadId }}
  ```

**Why Store in Sheets?**
- ✅ Easy to query from Telegram agent
- ✅ Searchable history
- ✅ No database setup needed
- ✅ Shareable and viewable

#### D. Conditional Router (If Node)
- **Type**: `n8n-nodes-base.if`
- **Version**: 2.3
- **Function**: Routes emails based on priority
- **Logic**:
  ```
  IF AI_Summary contains "High"
  OR Full_Email_Content equals "High"
  THEN send to Telegram
  ELSE skip notification
  ```

**Why This Design?**
- Only important emails trigger notifications
- Reduces Telegram spam
- Still stores everything for later query

#### E. Telegram Notifier
- **Type**: `n8n-nodes-base.telegram`
- **Function**: Sends summary to user's Telegram chat
- **Input**: `{{ $json.AI_Summary }}`
- **Delivery**: Async, no blocking

---

### 2. **Query/Chat Pipeline**

#### A. Telegram Trigger
- **Type**: `n8n-nodes-base.telegramTrigger`
- **Function**: Listens for user messages in Telegram
- **Webhook**: Sets up reverse webhook for instant messages
- **Output**: Message text, chat ID, user info

**Message Object:**
```javascript
{
  "message": {
    "text": "Who emailed me about budgets?",
    "chat": { "id": 1345923416 },
    "from": { "username": "user123" },
    "date": 1718356200
  }
}
```

#### B. Simple Memory (LangChain)
- **Type**: `@n8n/n8n-nodes-langchain.memoryBufferWindow`
- **Version**: 1.4
- **Function**: Remembers conversation history
- **Config**:
  - Session ID: User's chat ID
  - Context window: Last 10 messages
- **Why?**: AI can reference previous messages in same conversation

**Memory Flow:**
```
Message 1: "Hi"
Message 2: "What's new?" → AI remembers Message 1
Message 3: "Tell me more" → AI has context from 1 & 2
```

#### C. Google Sheets Tool (Query)
- **Type**: `n8n-nodes-base.googleSheetsTool`
- **Function**: Searches email database
- **Connection**: Used by Explainer Agent as a tool
- **Why?**: Agent can fetch real data when answering questions

#### D. OpenAI Chat Model
- **Type**: `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Version**: 1.3
- **Function**: Powers the Explainer Agent
- **Role**: Central "brain" for query handling

#### E. Explainer Agent
- **Type**: `@n8n/n8n-nodes-langchain.agent`
- **Version**: 3.1
- **Function**: Intelligent Q&A about emails
- **Tools Available**:
  - Google Sheets lookup
  - Memory buffer (conversation history)
  - OpenAI chat model

**System Prompt:**
```
You're a chill email assistant. Answer questions about emails
from the user's database in 1-2 sentences max.

Rules:
1. KEEP IT SHORT
2. USE THE EMAIL DATABASE
3. HANDLE GREETINGS NATURALLY
4. NEVER INVENT DATA
```

**Example Interactions:**
```
User: "Who sent the latest email?"
Agent: Searches Sheets → "That was Sarah from Sales."

User: "Tell me more about that"
Agent: Uses memory + Sheets → "She wants Q3 feedback by Friday."

User: "Hey, how are you?"
Agent: Chats normally → "I'm good! What's up?"
```

#### F. Telegram Response Sender
- **Type**: `n8n-nodes-base.telegram`
- **Function**: Sends agent response back to user
- **Input**: `{{ $json.output }}`

---

## Data Flow Diagrams

### Email Processing (Top-to-Bottom)

```
                    📨 New Email
                        │
                        ▼
                  [Gmail Trigger]
                        │
         ┌──────────────┴──────────────┐
         │                             │
         ▼                             ▼
    [Extract Data]              [Extract Data]
    - From                       - Subject
    - Subject                    - Snippet
    - Snippet                    - threadId
    - threadId
         │
         └─────────────┬─────────────┘
                       │
                       ▼
            [Summary Agent - OpenAI]
            (Creates 4-section summary)
                       │
                       ▼
         ┌─────────────────────────────┐
         │ [Google Sheets Append Row]  │
         │ Store: Email + Summary      │
         └─────────────┬───────────────┘
                       │
                       ▼
              [If - Check Priority]
              Contains "High"?
         ┌─────────────┬─────────────┐
         │ YES         │ NO          │
         ▼             ▼             
    [Send to       [End]
    Telegram]
         │
         ▼
    ✅ Done
```

### Query Processing (Left-to-Right)

```
      👤 User Types in Telegram
              │
              ▼
      [Telegram Trigger]
      (Webhook listening)
              │
      ┌───────┴────────┬──────────────┐
      │                │              │
      ▼                ▼              ▼
   [Message]    [Memory]       [Sheets Tool]
   "Who..."     (10 msgs)      (Available to Agent)
      │                │              │
      └────────┬───────┴──────────────┘
               │
               ▼
    [Explainer Agent - LangChain]
    Powered by OpenAI Chat Model
    (Decides: Search Sheets? Remember history? Chat?)
               │
               ▼
    [Agent Makes Decision]
    1. Understands question
    2. Searches Sheets if needed
    3. Uses memory for context
    4. Formulates response
               │
               ▼
      [OpenAI Generates Response]
      (1-2 sentences, conversational)
               │
               ▼
      [Telegram Sender]
               │
               ▼
      📤 Response sent to user
```

---

## Connection Map

```
Gmail Trigger
    ↓ (main)
Summary agent
    ↓ (main)
Append row in sheet
    ↓ (main)
If
    ├─ ✓ branch → Send a text message
    └─ ✗ branch → (nothing)

─────────────────────────────────

Telegram Trigger
    ↓ (main)
explainer Agent
    ↓ (main)
Send a text message1

explainer Agent ← uses ← OpenAI Chat Model
explainer Agent ← uses ← Get row(s) in sheet
explainer Agent ← uses ← Simple Memory
```

---

## Node Dependencies & Execution Order

```
Level 0 (Triggers - Always listening):
  - Gmail Trigger
  - Telegram Trigger

Level 1 (AI Processing):
  - Summary agent (waits for Gmail Trigger)
  - explainer Agent (waits for Telegram Trigger)

Level 2 (AI Models/Tools):
  - OpenAI (used by Summary agent)
  - OpenAI Chat Model (used by Explainer Agent)
  - Get row(s) in sheet (used by Explainer Agent)
  - Simple Memory (used by Explainer Agent)

Level 3 (Storage & Output):
  - Append row in sheet (waits for Summary agent)
  - If (waits for Append row in sheet)
  - Send a text message (waits for If)
  - Send a text message1 (waits for Explainer Agent)
```

---

## Error Handling

### What if Gmail Trigger Fails?
- Connection issue? → Retries on next poll
- Auth expired? → Manual re-authentication needed
- No new emails? → Skips (no error)

### What if OpenAI API Fails?
- Rate limited? → n8n queues & retries
- Invalid key? → Workflow fails, manual fix needed
- No credits? → Error stops workflow, monitor spending

### What if Google Sheets is Down?
- Email still processed by Summary agent
- Append fails → Data loss for that email
- User sees notification error in Telegram

### What if Telegram Can't Deliver?
- Network issue? → Telegram retries
- Invalid chat ID? → Error logged
- User blocked bot? → Silent fail

---

## Performance Considerations

### Polling Frequency
- **Current**: Every 1 minute
- **Impact**: 1,440 API calls/day to Gmail
- **Cost**: Minimal (Gmail is free)
- **Alternative**: Use webhook instead (faster, fewer calls)

### Concurrent Executions
- **Email + Query at same time**: Handled independently
- **Multiple emails/queries**: Queued and processed sequentially
- **Bottleneck**: OpenAI API rate limits

### Data Growth
- **1 year of emails**: ~250,000 rows in Google Sheets
- **Search time**: Still fast (Sheets handles millions)
- **Recommendation**: Archive old rows yearly

---

## Security & Privacy

### API Keys (Never in Code)
- Stored in n8n credentials vault
- Encrypted at rest
- Never exposed in logs
- Use `.gitignore` to prevent accidental commits

### Data Privacy
- Email content stored in Google Sheets (user's account)
- No external database
- User has full control
- Data accessible only to user's account

### Telegram Messages
- Sent over Telegram's encrypted channels
- Bot can't access user's other conversations
- Chat ID is private
- Token rotation recommended quarterly

---

## Scalability & Future Considerations

### Current Limits
- Single Gmail account
- Single Google Sheet
- Single Telegram chat
- ~100 emails/day (sustainable)

### To Scale Up
1. **Multi-account**: Duplicate workflow per account
2. **Database**: Replace Google Sheets with PostgreSQL
3. **Slack**: Add Slack Trigger + Sender
4. **Advanced AI**: Upgrade to GPT-4 for better summaries

### Cost Estimates
- Gmail: $0 (free)
- Google Sheets: $0 (free tier)
- Telegram: $0 (free)
- OpenAI: ~$0.01-0.05 per email
- n8n: $0 self-hosted, or $25+/month cloud

---

## Debugging Tips

### Enable Logging
1. In n8n, go to Settings → Logging
2. Set to "Debug" level
3. Re-run workflow
4. Check full logs for details

### Test Individual Nodes
1. Click on a node
2. Click "Test" button
3. Provide test data
4. See output in detail

### Monitor Executions
1. Click "Executions" in workflow
2. See success/failure history
3. Click each to inspect
4. Check error messages

---

## Related Documentation

- **[Setup Guide](./SETUP.md)** - How to install & configure
- **[README](./README.md)** - Features & overview
- **[n8n Docs](https://docs.n8n.io/)** - n8n platform documentation
- **[OpenAI API Docs](https://platform.openai.com/docs/)** - AI model info

---

*Architecture version 1.0 | Last updated June 2026*
