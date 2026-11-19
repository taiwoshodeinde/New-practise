# Gmail Setup Guide for FinMail Agent

Complete step-by-step guide, from zero to a working Gmail connection | October 24, 2026

Google changes its console wording from time to time, so menu names may differ slightly. The flow below follows Google's current Gmail Python quickstart (Google Auth platform > Branding, Audience, Data Access, Clients).

## What you will have at the end

| Item | Purpose |
| --- | --- |
| A Google Cloud project with the Gmail API enabled | Gives your app permission to call Gmail |
| `credentials.json` | Identifies your app. This is `GOOGLE_CREDENTIALS_PATH` |
| `token.json` | Your approval for the app to use your mailbox. This is `GOOGLE_TOKEN_PATH` |
| `get_gmail()`, `fetch_emails()`, `send_gmail_reply()` | Working code to plug into `whatsapp_email_agent.py` |

## Before you start

- A Gmail account (the mailbox the agent will read).
- Python 3.10 or later and a computer with a web browser for the one-time approval step.
- About 20 minutes.
- Recommended: make the work happen in a **new, separate** Google Cloud project, not an existing one.

## Part 1: Create a Google Cloud project

1. Open https://console.cloud.google.com and sign in with the Google account you will use for the project (ideally the same Gmail account).
2. Accept the terms if prompted.
3. Click the project picker at the top of the page, then **New project**.
4. Name it `FinMail Agent`. Leave Organization or Location as default (for a personal Gmail it shows No organization).
5. Click **Create**, wait a few seconds, then choose the new project in the picker.
6. Confirm the project name appears in the top bar. Everything below must be done in this project.

## Part 2: Enable the Gmail API

1. Open the menu (three lines), then **APIs & Services > Library**.
2. Search for `Gmail API` and open it.
3. Click **Enable**.
4. Wait for the API page to load (Overview, Metrics, Quotas tabs). If you later see a 403 saying the API is not enabled, come back and check this step, and wait a few minutes.

## Part 3: Configure Google Auth platform (the consent screen)

1. Open the menu, then **Google Auth platform**. If it says it is not configured yet, click **Get Started**. The Project configuration wizard opens.
2. **App Information:** App name `FinMail Agent`. User support email: choose your email. Click **Next**.
3. **Audience:**
   - Choose **External** if you use a normal `@gmail.com` account. External starts in testing mode and is limited to test users you list.
   - Choose **Internal** only if you belong to a Google Workspace organization and every user is inside it. Internal needs no verification. Click **Next**.
4. **Contact Information:** enter an email address for project notices. Click **Next**.
5. **Finish:** tick the box agreeing to the Google API Services User Data Policy. Click **Continue**, then **Create**.

## Part 4: Add yourself as a test user (External only)

This step is the most commonly missed. Without it, Google blocks you with an access\_denied error.

1. In **Google Auth platform**, open **Audience**.
2. Confirm Publishing status shows **Testing** and User type shows **External**.
3. Under **Test users**, click **Add users**.
4. Enter the exact Gmail address whose mail the agent will read. Click **Save**.

## Part 5: Add the Gmail scope (Data Access)

1. In **Google Auth platform**, open **Data Access**.
2. Click **Add or remove scopes**.
3. Find **Gmail API** in the list and tick `.../auth/gmail.readonly` (View your email messages and settings). If it is not listed, paste `https://www.googleapis.com/auth/gmail.readonly` into the manual scope box and click Add to table.
4. Click **Update**, then **Save**.
5. Start with read-only. You will add the send scope later in Part 11, only when you are ready to let the agent send approved replies.

## Part 6: Create the OAuth client and download credentials.json

1. In **Google Auth platform**, open **Clients**.
2. Click **Create client**.
3. **Application type:** choose **Desktop app** (not Web application). It is the last item in the dropdown, below Web application, Android, Chrome Extension, iOS and TVs and Limited Input devices. A wrong type causes a redirect\_uri\_mismatch error later.
4. Name: `FinMail Desktop`. Click **Create**.
5. In the confirmation dialog click **Download JSON**.
6. Rename the downloaded file (it has a long name starting with `client_secret_`) to `credentials.json`.
7. Treat this file as a secret. Do not email it, upload it to a public place or commit it to git.

**If a pink banner mentions a potential Acceptable Use Policy violation:** Google shows it when it flags a project you own, which may be a different project from this one. Click **Read more** to see which project and why, and follow the instructions on the notice. Do this before relying on the new project, because Google can suspend API access for flagged projects.

## Part 7: Prepare your project folder

1. Create a folder and move the file into it:

```
mkdir finmail-agent
cd finmail-agent
# move credentials.json here
```

2. Create a virtual environment and install the libraries:

```
python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install --upgrade google-api-python-client google-auth-httplib2 google-auth-oauthlib
pip install requests openai
```

3. Create a `.gitignore` containing at least:

```
credentials.json
token.json
agent_state.json
venv/
.env
```

## Part 8: Authorize once and create token.json

1. Create `authorize.py` in the folder:

```python
from google_auth_oauthlib.flow import InstalledAppFlow

SCOPES = ['https://www.googleapis.com/auth/gmail.readonly']

flow = InstalledAppFlow.from_client_secrets_file('credentials.json', SCOPES)
creds = flow.run_local_server(port=0)   # opens your browser

with open('token.json', 'w') as f:
    f.write(creds.to_json())
print('Saved token.json')
```

2. Run it on a computer with a browser: `python authorize.py`.
3. Your browser opens. Choose the Gmail account you added as a test user.
4. You will see **Google hasn't verified this app**. This is expected for your own app. Click **Advanced**, then **Go to FinMail Agent (unsafe)**.
5. Review the permission (view your email messages and settings) and click **Continue** or **Allow**.
6. The browser shows a short success message. Return to the terminal: it prints `Saved token.json`.
7. Check that `token.json` exists. It contains a refresh token, which lets the agent keep working without asking you again.

If you run the agent on a remote server, do this step on your laptop and copy the file across (Part 12). A server without a browser cannot complete it.

## Part 9: Test the connection

Create `test_gmail.py`:

```python
import os
from google.oauth2.credentials import Credentials
from google.auth.transport.requests import Request
from googleapiclient.discovery import build

SCOPES = ['https://www.googleapis.com/auth/gmail.readonly']

creds = Credentials.from_authorized_user_file('token.json', SCOPES)
if not creds.valid and creds.refresh_token:
    creds.refresh(Request())

gmail = build('gmail', 'v1', credentials=creds)
profile = gmail.users().getProfile(userId='me').execute()
print('Connected as', profile['emailAddress'])

res = gmail.users().messages().list(userId='me', maxResults=5).execute()
print('Messages found:', len(res.get('messages', [])))
```

Run `python test_gmail.py`. You should see your address and a message count. If you see an error, go to the troubleshooting table in Part 14.

## Part 10: Real code for the agent (read access)

Put this in `gmail_service.py`. It replaces the `fetch_emails()` stub in `whatsapp_email_agent.py`.

```python
import base64
import html
import os
import re
from email.utils import parseaddr

from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build

SCOPES = ['https://www.googleapis.com/auth/gmail.readonly']


def get_gmail():
    path = os.environ['GOOGLE_TOKEN_PATH']
    creds = Credentials.from_authorized_user_file(path, SCOPES)
    if not creds.valid:
        if creds.expired and creds.refresh_token:
            creds.refresh(Request())
            with open(path, 'w') as f:
                f.write(creds.to_json())      # keep the refreshed token
        else:
            raise RuntimeError('Gmail token invalid: re-run authorize.py')
    return build('gmail', 'v1', credentials=creds, cache_discovery=False)


def _header(headers, name):
    for h in headers:
        if h['name'].lower() == name.lower():
            return h['value']
    return ''


def _find_part(part, mime):
    data = part.get('body', {}).get('data')
    if part.get('mimeType') == mime and data:
        return base64.urlsafe_b64decode(data).decode('utf-8', 'replace')
    for sub in part.get('parts', []) or []:
        found = _find_part(sub, mime)
        if found:
            return found
    return None


def _body(payload):
    text = _find_part(payload, 'text/plain')
    if text:
        return text
    page = _find_part(payload, 'text/html')
    if page:
        page = re.sub(r'(?is)<script.*?</script>|<style.*?</style>', ' ', page)
        page = re.sub(r'<[^>]+>', ' ', page)
        return ' '.join(html.unescape(page).split())
    return ''


def fetch_emails(query, max_results=25):
    svc = get_gmail()
    resp = svc.users().messages().list(
        userId='me', q=query, maxResults=max_results).execute()
    emails = []
    for m in resp.get('messages', []):
        msg = svc.users().messages().get(
            userId='me', id=m['id'], format='full').execute()
        hdr = msg['payload']['headers']
        sender = _header(hdr, 'From')
        reply_to = _header(hdr, 'Reply-To') or sender
        emails.append({
            'id': msg['id'],
            'thread_id': msg['threadId'],
            'from': sender,
            'reply_to': parseaddr(reply_to)[1],
            'subject': _header(hdr, 'Subject'),
            'message_id_header': _header(hdr, 'Message-ID'),
            'body': _body(msg['payload']),
        })
    return emails
```

Then in `whatsapp_email_agent.py` import it and call `fetch_emails(GMAIL_QUERY)` inside the existing `fetch_emails()` stub. Gmail search syntax works in the query, for example `newer_than:2d (invoice OR receipt OR statement)`.

## Part 11: Enable sending (do this last, after reading works)

Only do this when classification and drafts look right and you want approved replies to go out.

1. In **Google Auth platform > Data Access**, add the scope `https://www.googleapis.com/auth/gmail.send`. Save.
2. In `authorize.py` and `gmail_service.py`, change SCOPES to:

```python
SCOPES = [
    'https://www.googleapis.com/auth/gmail.readonly',
    'https://www.googleapis.com/auth/gmail.send',
]
```

3. Delete the old `token.json` and run `python authorize.py` again. Approve the new permission screen. Scope changes always need a fresh token.
4. Add this function to `gmail_service.py`:

```python
from email.message import EmailMessage


def send_reply(draft):
    svc = get_gmail()
    msg = EmailMessage()
    msg['To'] = draft['to']
    msg['Subject'] = draft['subject']
    msg['In-Reply-To'] = draft['in_reply_to']
    msg['References'] = draft['in_reply_to']
    msg.set_content(draft['body'])
    raw = base64.urlsafe_b64encode(msg.as_bytes()).decode()
    return svc.users().messages().send(
        userId='me',
        body={'raw': raw, 'threadId': draft['thread_id']},
    ).execute()
```

5. Call `send_reply(draft)` from `send_gmail_reply()` in the agent, and only from the `send N` approval handler.
6. Test with a draft addressed to yourself first. Keep `DRY_RUN` on until that works. Keep a `SEND_ENABLED` kill switch, as the SRS requires.

## Part 12: Move to your server

1. Copy `credentials.json`, `token.json` and your code to the server (for example with scp).
2. Restrict permissions: `chmod 600 credentials.json token.json`.
3. Set environment variables (names from the SRS):

| Variable | Example |
| --- | --- |
| GOOGLE\_CREDENTIALS\_PATH | /opt/finmail/credentials.json |
| GOOGLE\_TOKEN\_PATH | /opt/finmail/token.json |
| WA\_API\_TOKEN | from WhatsApp > Settings > Agents |
| OPENAI\_API\_KEY | your OpenAI key |
| OPENAI\_MODEL | the model you chose |

4. The app must be able to write to `token.json`, because refreshed tokens are saved there.
5. Run the agent under systemd, Docker or pm2 so it restarts after a crash.

## Part 13: Token expiry and moving to production

- While an External app is in **Testing** status, Google can expire refresh tokens after about 7 days (check Google's current policy). If the agent suddenly reports an invalid token, run `authorize.py` again and copy the new `token.json`.
- To avoid this for personal use, open **Google Auth platform > Audience** and click **Publish app** to move it to production. The app stays unverified, so you still click through the warning once, and Google limits unverified apps to 100 users, which is fine for a single owner.
- Restricted Gmail scopes require a formal Google verification only if you distribute the app to other people. For your own mailbox you do not need it.
- If you use Workspace and chose **Internal**, none of this applies.

## Part 14: Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| access\_denied or app blocked at sign-in | Your account is not a test user | Part 4: add the exact Gmail address |
| redirect\_uri\_mismatch | Client type is not Desktop app | Recreate the client as Desktop app (Part 6) |
| 403 accessNotConfigured or Gmail API has not been used | API not enabled in this project | Part 2; wait a few minutes; confirm correct project |
| 403 insufficient authentication scopes | Token lacks the scope you now use | Add scope in Data Access, delete token.json, re-run authorize.py |
| invalid\_grant, token expired or revoked | Testing-mode expiry, password change, or access revoked | Re-run authorize.py, copy new token.json |
| No refresh token in token.json | Previously approved without offline access | Revoke the app at myaccount.google.com/permissions, delete token.json, authorize again |
| Browser does not open on a server | No display | Run authorize.py on your laptop and copy token.json |
| Google hasn't verified this app | Normal for personal apps | Advanced > Go to FinMail Agent |
| 429 or rateLimitExceeded | Too many calls | Back off and retry; limit to 25 messages per check |
| File not found for credentials.json | Wrong path or name | Check the downloaded file was renamed and the env var path is absolute |
| Pink Acceptable Use Policy banner in Google Cloud | Google flagged a project you own, possibly a different one | Click Read more, find which project is named, follow the notice; use a fresh project for this agent |

## Part 15: Security checklist

- [ ] credentials.json and token.json are in .gitignore and never shared.
- [ ] File permissions are 600 on the server.
- [ ] Only gmail.readonly is enabled until you are ready to send.
- [ ] gmail.send is called from one function, reachable only from the owner's send command.
- [ ] A kill switch (SEND\_ENABLED) and dry-run mode exist.
- [ ] You know how to revoke access: https://myaccount.google.com/permissions, remove FinMail Agent.
- [ ] Email bodies are not written to logs.

## Final checklist

- [ ] New project created and selected
- [ ] Gmail API enabled
- [ ] Consent screen configured (External or Internal)
- [ ] Your Gmail added as test user
- [ ] gmail.readonly scope added in Data Access
- [ ] Desktop OAuth client created, credentials.json downloaded
- [ ] Libraries installed in a virtual environment
- [ ] authorize.py run, token.json created
- [ ] test\_gmail.py prints your address
- [ ] fetch\_emails wired into the agent
- [ ] Later: gmail.send added, token recreated, send\_reply tested on yourself
