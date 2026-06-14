# ContextMail Setup Guide 🚀

Complete step-by-step instructions to get ContextMail running on your system.

---

## Prerequisites Checklist

Before you start, make sure you have:

- [ ] An **n8n instance** (free cloud at [n8n.io](https://n8n.io) or self-hosted)
- [ ] **OpenAI API key** (get one at [platform.openai.com](https://platform.openai.com))
- [ ] **Gmail account** (with 2-factor authentication enabled)
- [ ] **Telegram account** (with a bot created)
- [ ] **Google account** (for Sheets & OAuth2)

---

## Step 1: Create a Telegram Bot

### 1.1 Open Telegram and find BotFather

```
1. Open Telegram app
2. Search for "BotFather" (official Telegram bot manager)
3. Click /start
```

### 1.2 Create a new bot

```
1. Type: /newbot
2. Give your bot a name: "ContextMail"
3. Give it a username: "contextmail_bot" (must be unique)
4. Copy the API token (looks like: 123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11)
```

### 1.3 Get your Chat ID

```
1. Still in Telegram, search for "userinfobot"
2. Click /start
3. Copy your chat ID (9-digit number)
```

**Save these:**
- Bot Token: `123456:ABC-...`
- Chat ID: `1345923416`

---

## Step 2: Create a Google Sheet for Email Database

### 2.1 Create a new spreadsheet

```
1. Go to Google Sheets: https://sheets.google.com
2. Click "New" → "Spreadsheet"
3. Name it: "ContextMail Database"
```

### 2.2 Set up columns

Create these column headers in Row 1:

| A | B | C | D | E | F |
|---|---|---|---|---|---|
| Sender_Email | Subject | Full_Email_Content | AI_Summary | Received_Time | Thread_ID |

### 2.3 Copy your Spreadsheet ID

```
The URL looks like:
https://docs.google.com/spreadsheets/d/1-CfA5OBHBsJWLAdNhL6yzKVxEyb8mYn0pX9rj6DC2SA/edit

Copy this part: 1-CfA5OBHBsJWLAdNhL6yzKVxEyb8mYn0pX9rj6DC2SA
```

---

## Step 3: Set Up n8n Credentials

### 3.1 OpenAI API Key

```
1. Go to https://platform.openai.com/api/keys
2. Click "Create new secret key"
3. Copy the key immediately (you can't see it again)
4. In n8n: Create credential → OpenAI API → Paste key
```

### 3.2 Gmail OAuth2

```
1. In n8n: Create credential → Gmail OAuth2
2. Follow the popup to connect your Gmail account
3. Allow access when prompted
```

### 3.3 Telegram Bot Token

```
1. In n8n: Create credential → Telegram API
2. Paste your bot token from Step 1
```

### 3.4 Google Sheets OAuth2

```
1. In n8n: Create credential → Google Sheets OAuth2
2. Follow popup to connect your Google account
3. Allow access to Google Sheets
```

---

## Step 4: Import and Configure the Workflow

### 4.1 Download the workflow file

Download `workflow.json` from this GitHub repository.

### 4.2 Import into n8n

```
1. In n8n, click "Workflows" → "Import"
2. Upload the workflow.json file
3. Click "Import"
```

### 4.3 Configure Gmail Trigger

```
Node: "Gmail Trigger"
1. Click on the node
2. In credentials, select your Gmail OAuth2 connection
3. Under "Poll Times", verify it says "Every Minute"
4. Leave Filters empty (or customize if needed)
```

### 4.4 Configure Summary Agent

```
Node: "Summary agent"
1. Verify credentials are set to your OpenAI key
2. Review the prompt text (optional: customize tone/style)
3. Keep model as "gpt-5-mini"
```

### 4.5 Configure Google Sheets (Append)

```
Node: "Append row in sheet"
1. Select credentials: Google Sheets OAuth2
2. Under "Document ID", paste your Spreadsheet ID
3. Under "Sheet Name", select "Summary chat history"
4. Column mappings should auto-populate
```

### 4.6 Configure Telegram Nodes

```
Node: "Send a text message"
1. Credentials: Select your Telegram bot
2. Chat ID: Paste your chat ID (1345923416)

Node: "Send a text message1"
1. Same settings as above
```

### 4.7 Configure Google Sheets Tool

```
Node: "Get row(s) in sheet in Google Sheets"
1. Credentials: Google Sheets OAuth2
2. Document ID: Paste your Spreadsheet ID
3. Sheet Name: Select "Summary chat history"
```

### 4.8 Configure Telegram Trigger

```
Node: "Telegram Trigger"
1. Credentials: Select your Telegram bot
2. Leave other settings as default
```

---

## Step 5: Test the Workflow

### 5.1 Activate the workflow

```
1. Click the big red toggle in top-right: "Activate"
2. Confirm the prompt
3. Wait for it to turn green
```

### 5.2 Test email summarization

```
1. Send an email to your Gmail account
2. Give it 1-2 minutes to process
3. Check your Telegram chat - you should see a summary!
4. Check your Google Sheet - new row should appear
```

### 5.3 Test interactive agent

```
1. In Telegram, send a message to your bot:
   "Hi, how are you?"
2. Bot should respond conversationally
3. Try: "Who emailed me latest?"
4. Bot should find info from Google Sheets
```

### 5.4 Debug if needed

If something doesn't work:

```
1. Click "Executions" in the workflow
2. Look at the latest run
3. Check which node failed (red X)
4. Click the failed node to see the error
5. Common issues:
   - Wrong credentials (re-check Step 3)
   - Invalid Spreadsheet ID (copy-paste carefully)
   - Telegram chat ID wrong (verify with userinfobot)
```

---

## Step 6: Customize (Optional)

### Adjust polling frequency
- Save API calls by checking Gmail every 5 minutes instead of 1
- Node: "Gmail Trigger" → Change "Poll Times" to "Every 5 Minutes"

### Change summary format
- Node: "Summary agent" → Edit the prompt text
- Make it shorter, add emojis, change tone

### Adjust priority routing
- Node: "If" → Modify the conditions
- Example: Only send High priority emails to Telegram

### Filter certain emails
- Node: "Gmail Trigger" → Add Filters
- Example: Skip marketing emails, only process work inbox

---

## Troubleshooting

### "Gmail Trigger says 'No data'"
- Check Gmail is authenticated
- Give it 2-3 minutes (polling every minute)
- Send yourself a test email

### "API key error"
- Verify OpenAI key is valid (not expired)
- Check you're not running out of credits
- Test on OpenAI website first

### "Google Sheets shows permission error"
- Re-authenticate Google Sheets OAuth2
- Make sure you gave permission to access Sheets
- Verify Spreadsheet ID is correct

### "Telegram not sending"
- Check chat ID is correct (use @userinfobot)
- Verify bot token is right
- Make sure bot has permission to send messages

### "Summaries look weird"
- The prompt might need tuning
- Edit it in "Summary agent" node
- Try adding examples or changing tone

---

## Success! 🎉

You're now running ContextMail! 

**Next steps:**
1. Monitor emails for 1-2 days to see summaries
2. Adjust the summary prompt if needed
3. Customize priority rules for your workflow
4. Consider future improvements (Slack, daily digests, etc.)

---

## Getting Help

- **n8n Docs**: https://docs.n8n.io/
- **OpenAI API Docs**: https://platform.openai.com/docs/
- **GitHub Issues**: Open an issue on the repo
- **Community**: Ask on n8n forums or Discord

Happy automating! 🚀
