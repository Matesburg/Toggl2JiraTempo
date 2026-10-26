# Setup Guide - Toggl to Jira Tempo Sync Tool

## Quick Start (5 minutes)

### 1. Install Python Dependencies

```bash
pip install -r requirements.txt
```

### 2. Create & Configure `.env` File

Copy the example template:
```bash
cp .env.example .env
```

Edit `.env` and fill in your credentials:
```env
# Toggl API Configuration
TOGGL_API_TOKEN=YOUR_TOGGL_API_TOKEN_HERE
TOGGL_WORKSPACE_ID=YOUR_WORKSPACE_ID_HERE
TOGGL_CLIENT_ID=YOUR_TOGGL_CLIENT_ID_HERE

# Jira Cloud Configuration
JIRA_URL=https://your-domain.atlassian.net
JIRA_EMAIL=your-email@example.com
JIRA_API_TOKEN=YOUR_JIRA_API_TOKEN_HERE

# Sync Configuration
FIRST_SYNC_DATE=2024-01-01
LOG_LEVEL=INFO
```

### 3. Run First Sync

```bash
python main.py
```

---

## Detailed Setup Instructions

### Getting Toggl API Token

1. Open Toggl: https://track.toggl.com/app
2. Click your **Profile** icon (top-right)
3. Select **Settings**
4. Scroll to **API Token** section
5. Copy the token (it's a long hex string)

**Note:** Keep this token secret - it gives access to your time entries!

### Finding Toggl Workspace ID

**Quick Method - From Dropdown:**
1. Go to https://track.toggl.com/app
2. Click the **workspace name** at the top-left (e.g., "My Workspace")
3. Look for your workspace ID in the dropdown - it's a 7-8 digit number
4. Example: `12345678` ← that's your Workspace ID

**Most Reliable Method - Using PowerShell:**

Since Toggl's UI can be inconsistent, here's the guaranteed way:

1. First, get your Toggl API token (from https://track.toggl.com/app/profile)
2. Open PowerShell and run:
```powershell
$token = "YOUR_TOGGL_API_TOKEN_HERE"
$auth = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes("${token}:api_token"))
$response = Invoke-RestMethod -Uri "https://api.track.toggl.com/api/v8/me" `
  -Headers @{Authorization = "Basic $auth"}
$response.workspaces[0].id
```
3. This prints a number like `12345678` - that's your Workspace ID!

**Browser DevTools Method:**
1. Go to https://track.toggl.com/app
2. Open Developer Tools (Press F12)
3. Go to **Network** tab
4. Refresh page (F5)
5. Click on any Toggl API request
6. Look in **Request Headers** for `Workspace-ID` header
7. Copy that number

### Getting Jira API Token

1. Open Jira: https://id.atlassian.com/manage-profile/security/api-tokens
2. Click **Create API token**
3. Name it "Toggl Sync" or similar
4. Copy the token immediately (you can't view it again)

**Scopes needed:**
- `write:issue-worklog:jira` - Create/update worklogs

### Jira Cloud URL

Use your domain:
- If you access Jira at: `https://mycompany.atlassian.net/`
- Then `JIRA_URL` = `https://mycompany.atlassian.net`

---

## Verification Steps

### Test Toggl Connection

```bash
# Replace with your token
curl -H "Authorization: Basic $(echo -n 'YOUR_TOKEN:api_token' | base64)" \
     https://api.track.toggl.com/api/v8/me
```

Should return your Toggl profile info.

### Test Jira Connection

```bash
# Replace with your email and token
curl -H "Authorization: Basic $(echo -n 'email@company.com:YOUR_JIRA_TOKEN' | base64)" \
     https://your-domain.atlassian.net/rest/api/3/myself
```

Should return your Jira profile info.

### Verify Database Dependencies

The tool uses SQLite (built-in with Python), so no additional setup needed!

---

## First Sync Configuration

### Choosing Start Date

For `FIRST_SYNC_DATE` in `.env`:

**Option 1: Sync all entries since date**
```env
FIRST_SYNC_DATE=2024-01-01
```
On first run, you'll be asked to confirm this date.

**Option 2: Let script ask on first run**
Just leave the default, and the script will prompt you.

### Client Filter

The tool filters by client **ID**, not by name, since client names can be renamed later while the ID stays stable:
```env
TOGGL_CLIENT_ID=12345678
```

See the README's "Finding Your API Tokens" section for how to look up your client ID.

If entries aren't showing up, check:
1. Is the client ID correct?
2. Are the projects assigned to this client?

---

## Troubleshooting Setup

### "Missing required configuration"

**Error:**
```
❌ Missing required configuration: TOGGL_API_TOKEN, JIRA_API_TOKEN
```

**Fix:**
1. Create `.env` file: `cp .env.example .env`
2. Edit it with real values (not examples)
3. Don't use quotes around values

### "No module named 'requests'"

**Error:**
```
ModuleNotFoundError: No module named 'requests'
```

**Fix:**
```bash
pip install -r requirements.txt
```

### "Jira authentication failed"

**Symptoms:**
- Tool runs but fails on Jira connection test
- Error: "Jira authentication failed"

**Troubleshooting:**
1. Verify email is correct: `echo $JIRA_EMAIL`
2. Verify API token is correct (not password!)
3. Check token hasn't been revoked: https://id.atlassian.com/manage-profile/security/api-tokens
4. Ensure token has right scopes

**Test with curl:**
```bash
curl -u "your-email@example.com" https://your-domain.atlassian.net/rest/api/3/myself
# Enter API token as password when prompted
```

### "Toggl authentication failed"

**Symptoms:**
- Error when fetching Toggl entries
- "401 Unauthorized"

**Troubleshooting:**
1. Verify API token (not password!)
2. Check token hasn't expired
3. Get new token from: https://track.toggl.com/app/profile
4. Test with curl:
```bash
curl -H "Authorization: Basic $(echo -n 'YOUR_TOKEN:api_token' | base64)" \
     https://api.track.toggl.com/api/v8/me
```

### "No entries found"

**Symptoms:**
- Sync runs but finds 0 entries

**Troubleshooting:**
1. **Check client ID**: Does `TOGGL_CLIENT_ID` in `.env` match the client's ID in Toggl?
2. **Check date range**: Do you have entries in that date range?
3. **Check projects**: Are projects assigned to that client?
4. **Check project names**: Do they start with Jira issue key format?
   - Valid: `PROJ-123 Feature`
   - Invalid: `Feature PROJ-123` (wrong order)

### Logs Aren't Helpful

**To enable debug logging**, edit `.env`:
```env
LOG_LEVEL=DEBUG
```

Then run again and check `logs/sync_*.log` for detailed output.

---

## File Structure After Setup

```
Toggl2JiraTempo/
├── .env                    ← Your credentials (NOT in git!)
├── .env.example           ← Template (safe to commit)
├── .gitignore            ← Ignore .env, logs, db
├── requirements.txt       ← Python dependencies
├── main.py               ← Main entry point - RUN THIS
├── config.py             ← Load credentials
├── toggl_api.py          ← Toggl API client
├── jira_api.py           ← Jira API client
├── db.py                 ← SQLite database
├── sync_engine.py        ← Core sync logic
├── validator.py          ← Validation utilities
├── ui.py                 ← User interface
├── README.md             ← Full documentation
├── SETUP.md              ← This file
├── sync_state.db         ← SQLite db (auto-created on first run)
└── logs/                 ← Sync logs (auto-created)
    └── sync_20240115_143000.log
```

---

## Next Steps

1. ✓ Installed Python dependencies
2. ✓ Created and edited `.env` 
3. → Run `python main.py` to start first sync!

---

## Getting Help

1. **Check logs**: `ls logs/` - latest log file has detailed info
2. **Enable debug**: Set `LOG_LEVEL=DEBUG` in `.env`
3. **Test APIs manually**: Use curl commands above to verify credentials
4. **Read README.md**: Full documentation and troubleshooting

---

## Security Reminders

⚠️ **DO NOT:**
- Commit `.env` file to git
- Share your API tokens
- Expose `sync_state.db` (contains sync history)
- Post logs online without redacting tokens

✓ **DO:**
- Keep `.env` file locally only
- Regenerate tokens if accidentally exposed
- Add `.env` to `.gitignore` (already done)
- Keep logs private for troubleshooting
