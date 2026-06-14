# ContextMail 📧

An AI-powered email assistant that automatically summarizes incoming emails, stores context, and lets you query your email history via Telegram—without ever manually searching your inbox again.

---

## ✨ Features

- **Automatic Email Summarization** – AI-generated summaries with priority levels (High/Medium/Low)
- **Smart Storage** – Email data stored in Google Sheets for easy access
- **Priority Routing** – Important emails automatically sent to Telegram
- **Conversational Query** – Ask questions about past emails via Telegram chat
- **Context Memory** – AI remembers conversation history for smarter responses
- **No Manual Work** – Fully automated with n8n orchestration

---

## 🎯 What It Does

### Scenario 1: You receive an important email
```
📨 Email arrives → AI summarizes it → Stores in database → Sends to Telegram
"Hey! Your client just sent a proposal with a Friday deadline. Priority: High."
```

### Scenario 2: You want to find information
```
💬 You ask in Telegram: "Who emailed me about the Q4 budget?"
🤖 Assistant responds: "That was Sarah from Finance about the quarterly review."
```

---

## 🚀 Quick Start

### Prerequisites
- **n8n** instance (self-hosted or cloud)
- **OpenAI API key** (for summaries & AI agent)
- **Gmail account** with OAuth2 configured
- **Telegram bot token** & chat ID
- **Google Sheets** (for email database)

### Setup Steps

1. **Import the Workflow**
   - Open your n8n instance
   - Go to Workflows → Import
   - Upload `workflow.json`

2. **Configure Credentials**
   - Add your **OpenAI API key**
   - Connect your **Gmail OAuth2** account
   - Add your **Telegram bot token**
   - Connect your **Google Sheets** account

3. **Update Configuration**
   - In the "Summary agent" node, review the prompt (customizable)
   - Update the **Google Sheets document ID** in the "Append row in sheet" node
   - Update the **Telegram chat ID** to your personal chat
   - Set your **polling interval** in the Gmail Trigger (currently every minute)

4. **Activate the Workflow**
   - Toggle "Active" to turn on automation
   - Test by sending yourself an email

---

## 📋 How It Works

### Email Processing Pipeline

```
Gmail Trigger (every minute)
    ↓
Summary Agent (OpenAI creates 4-section summary)
    ↓
Append to Google Sheets (stores all email data)
    ↓
If Condition (checks priority level)
    ↓
Send Telegram (for High/Medium priority)
```

### Interactive Query Pipeline

```
Telegram Trigger (user sends message)
    ↓
Simple Memory (remembers conversation)
    ↓
Explainer Agent (asks: "Which email are they asking about?")
    ↓
Get Rows from Google Sheets (searches email database)
    ↓
OpenAI Chat Model (generates response)
    ↓
Send Telegram Response
```

---

## 🎨 Summary Format

Every email is summarized into 4 sections:

```
🎯 The Pitch
Quick intro: "Hey! Arjun just emailed about [topic]."

📋 The Details
Main points in 2-3 sentences.

⏰ The Timeline
Important dates or "No specific timeline—just a heads up."

🚨 Priority Level
High | Medium | Low + 1-sentence explanation
```

**Priority Rules:**
- **High** → Urgent, deadlines, work requests, client communications
- **Medium** → Important updates, non-urgent follow-ups
- **Low** → Newsletters, marketing, promotions, social notifications

---

## 📊 Google Sheets Database Schema

Your email data is stored with these columns:

| Column | Description |
|--------|-------------|
| `Sender_Email` | Email address of sender |
| `Subject` | Email subject line |
| `Full_Email_Content` | Email snippet/preview |
| `AI_Summary` | 4-section summary from OpenAI |
| `Received_Time` | Timestamp email arrived |
| `Thread_ID` | Gmail thread ID (for context) |

---

## 🛠️ Customization

### Change the Summary Format
Edit the "Summary agent" node → Update the prompt to fit your style.

**Example:** Want emojis? Shorter summaries? Different sections? Just edit the text!

### Adjust Priority Rules
In the "If" node, modify the conditions:
- Check `AI_Summary` contains "High"
- Check `Full_Email_Content` equals certain keywords

### Adjust Polling Frequency
In the "Gmail Trigger" node:
- Change from "Every Minute" to "Every 5 Minutes" (saves API calls)
- Set specific times or intervals

### Filter Emails
In the "Gmail Trigger" node → Filters:
- Exclude certain senders
- Only process specific labels
- Ignore newsletters automatically

---

## ⚙️ Architecture

```
┌─────────────┐
│   Gmail     │
│   Inbox     │
└──────┬──────┘
       │
       ▼
┌─────────────────────────┐
│  n8n Workflow           │
│  ├─ Email Trigger       │
│  ├─ AI Summarizer       │
│  ├─ Data Storage        │
│  ├─ Priority Router     │
│  └─ Telegram Notifier   │
└──────┬──────────────────┘
       │
       ├──────────────────┬──────────────────┐
       ▼                  ▼                  ▼
┌──────────────┐  ┌─────────────────┐  ┌──────────────┐
│Google Sheets │  │   Telegram      │  │   OpenAI     │
│  Database    │  │   (Mobile UI)   │  │   (Brain)    │
└──────────────┘  └─────────────────┘  └──────────────┘
```

---

## 🤔 FAQ

**Q: Will this work with other email providers?**
A: Currently configured for Gmail. You can swap Gmail Trigger with Outlook, Yahoo, etc.

**Q: How much does this cost?**
A: Mainly OpenAI API costs (roughly $0.01-0.05 per email). Gmail & Google Sheets are free-tier friendly.

**Q: Can I use this for work and personal emails?**
A: Yes! Create multiple sheets or workflows for different email accounts.

**Q: What happens to old emails?**
A: They stay in Google Sheets forever (unless you delete rows). You can always query them!

**Q: Can I turn off Telegram notifications?**
A: Remove the "Send a text message" node or adjust the "If" conditions.

---

## 🚀 Future Improvements

- [ ] Slack integration instead of (or in addition to) Telegram
- [ ] Scheduled digest emails (daily/weekly summary)
- [ ] Smart folder organization (auto-labels based on summary)
- [ ] Sentiment analysis (detect urgent tone)
- [ ] Natural language search ("Find emails about budgets from last month")
- [ ] Follow-up reminders ("Reply to Sarah's email in 2 days")
- [ ] Custom priority rules per sender
- [ ] Dark mode for Telegram UI
- [ ] Export summaries to PDF/Word reports

---

## 📝 License

This project is open-source and available under the MIT License.

---

## 🙋 Support

Have questions or issues?
- Check the [n8n Documentation](https://docs.n8n.io/)
- Review [OpenAI API Docs](https://platform.openai.com/docs/)
- Open an issue on GitHub

---

## Screen Shot's
<img width="926" height="332" alt="Screenshot 2026-06-14 150510" src="https://github.com/user-attachments/assets/d5629c1d-8b97-456b-afc7-c6f655c6e5c5" />

<img width="614" height="290" alt="Screenshot 2026-06-14 150857" src="https://github.com/user-attachments/assets/865ec3f0-8719-4ae5-ba4c-f84e78b130ba" />

<img width="905" height="405" alt="Screenshot 2026-06-14 162517" src="https://github.com/user-attachments/assets/a661ede8-b357-49cf-87c8-a452af9c3c6f" />

---
## 👨‍💻 Author

Built as an AI automation project to save time on email management.

**Happy emailing! 🚀**
