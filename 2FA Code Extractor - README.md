# 2FA Code Extractor — n8n Workflow

A self-service workflow that lets users retrieve 2FA verification codes from a shared email account without needing direct access to the inbox.

---

## The Problem

Your team shares a single email address to sign up for platforms (e.g., Udemy, Netflix, Coursera). When a platform sends a verification code to that email, users can't access it directly — they'd have to message the admin and wait. This causes delays, and codes often expire before the admin responds.

## The Solution

This workflow gives users a simple form link. They fill in their name, and the workflow instantly searches the shared email inbox for the latest verification code and displays it on screen.

---

## How It Works

```
User fills form  -->  Timestamp added  -->  Request logged  -->  Gmail searched  -->  Code extracted  -->  Code displayed
```

1. **User opens the form link** and enters their name
2. **A timestamp is generated** and attached to the request data
3. **Request is logged** to a Google Sheet (name + timestamp for audit trail)
4. **Gmail is searched** for unread 2FA emails from the last 15 minutes
5. **The code is extracted** using pattern matching (supports 4-8 digit/alphanumeric codes)
6. **The code is displayed** directly on the form response page

The entire process takes a few seconds.

---

## Workflow Nodes

| # | Node | Purpose |
|---|------|---------|
| 1 | **Form Trigger** | Web form where users enter their name |
| 2 | **Prepare Log Data** | Attaches a timestamp to the form data before logging |
| 3 | **Log Request to Sheet** | Appends name + timestamp to Google Sheets for audit trail |
| 4 | **Gmail - Fetch Recent 2FA Emails** | Searches Gmail for unread verification emails (last 15 min) |
| 5 | **Extract 2FA Code** | Parses the email body/subject to find the code, builds the response |
| 6 | **Respond to User** | Displays the result to the user on the form page |

---

## Setup Instructions

### Prerequisites
- n8n instance (self-hosted or cloud)
- Gmail account connected to n8n (OAuth2)
- Google Sheets account connected to n8n (OAuth2)

### Step-by-Step

1. **Import the workflow**
   - In n8n, go to **Workflows > Import from File**
   - Select `2FA Code Extractor.json`

2. **Prepare your Google Sheet**
   - Create a new Google Sheet (or use an existing one)
   - Add exactly two column headers in row 1: `Full Name` and `Timestamp`
   - Headers must match exactly — case-sensitive, no extra spaces

3. **Configure the Log Request to Sheet node**
   - Open the **Log Request to Sheet** node
   - Select your Google Sheet from the **Document** dropdown
   - Select the sheet tab from the **Sheet** dropdown
   - Operation must be set to **Append Row**
   - Mapping mode must be set to **Auto Map Input Data**
   - After selecting the sheet, re-select both dropdowns to force n8n to refresh the column list

4. **Verify Gmail credentials**
   - Open the **Gmail - Fetch Recent 2FA Emails** node
   - Confirm your Gmail OAuth2 credential is connected
   - This must be the Gmail account that receives the 2FA codes

5. **Update the Respond to User node**
   - Open the **Respond to User** node
   - Click **Add Option** → **Response Headers**
   - Add header: Name = `Content-Type`, Value = `text/html`
   - This enables the styled HTML response page

6. **Test the workflow**
   - Click the **Form Trigger** node and copy the **Test URL**
   - Open the URL in a browser, enter a name, and submit
   - Trigger a real 2FA code from any platform, then submit the form again to verify extraction
   - Check your Google Sheet — both the name and timestamp should appear

7. **Activate for production**
   - Toggle the **Active** switch in the top-right corner
   - Copy the **Production URL** from the Form Trigger node
   - Share this URL with your users

---

## What Users See

### When a code is found (styled HTML page):
- Green bordered card with the code displayed large and bold
- Clear urgency message to use the code immediately
- Fallback instructions if the code doesn't work

### When no code is found (styled HTML page):
- Amber bordered card with a friendly message
- Two numbered steps explaining what to do next
- Admin contact suggestion at the bottom

---

## User Instructions (Share This With Your Team)

### How to get your verification code:

1. Go to the platform and sign up / log in using the shared email and password
2. When the platform says "Enter your verification code", **immediately** open this form link:
   `[YOUR PRODUCTION URL HERE]`
3. Enter your full name and click Submit
4. Your code will appear on screen — copy it and enter it on the platform
5. If no code is found, wait 30 seconds and submit the form again

### Important:
- Submit the form **right after** the platform asks for a code — codes expire in a few minutes
- If the code doesn't work, the platform may have sent a new one — submit the form again for the latest code

---

## Supported Platforms

The workflow works with **any platform** that sends verification codes via email, including but not limited to:

- Udemy, Coursera, LinkedIn Learning
- Netflix, Spotify, Disney+
- Google, Microsoft, Apple
- Social media (Instagram, Twitter/X, Facebook)
- Banking and fintech apps
- Any service that sends 4-8 digit verification codes

---

## Code Extraction Details

The extraction engine uses three strategies (in priority order):

1. **Keyword-adjacent matching** — Finds codes near words like "verification code", "OTP", "security code", "login code"
   - Example: "Your verification code is **482910**"
   - Example: "OTP: **5839**"

2. **Subject line matching** — Finds numeric codes in the email subject
   - Example: Subject: "**739201** is your login code"

3. **Standalone number matching** — Finds prominent 6-digit numbers in the email body
   - Example: Body contains just "**103847**"

### Supported code formats:
- 4 to 8 digits (numeric): `4829`, `482910`, `48291037`
- 4 to 8 characters (alphanumeric): `A8K2M1`, `XY482910`

---

## Gmail Search Query

The workflow searches for unread emails from the last 15 minutes using:

```
newer_than:15m (verification OR "verify your" OR "security code" OR "confirmation code" OR "login code" OR OTP OR "one-time" OR "2FA")
```

This catches emails from virtually all platforms that send 2FA codes.

---

## Audit & Logging

Every request is logged to Google Sheets automatically. The **Prepare Log Data** node generates a timestamp at the exact moment the form is submitted and passes it to the Sheets node.

| Column | Value | Notes |
|--------|-------|-------|
| `Full Name` | Name entered by the user | From the form |
| `Timestamp` | ISO 8601 datetime (e.g. `2026-03-11T14:56:46.123Z`) | Generated at time of submission |

The log is written **before** Gmail is searched — so every access attempt is recorded regardless of whether a code was found.

This helps you:
- Track who is requesting codes and when
- Identify if someone is submitting excessive requests
- Maintain a record for accountability

### Important setup notes for logging:
- The Google Sheets node operation must be **Append Row** (not Append or Update Row)
- Column headers in the sheet must match exactly: `Full Name` and `Timestamp`
- After adding new columns to the sheet, re-select the document and sheet dropdowns in the node to refresh n8n's column cache

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| "Code not found" every time | Gmail credentials may not be for the shared email | Verify the Gmail node is connected to the correct account |
| Code returned is wrong/random | Another email matched the search keywords | Check the Gmail node output to see which emails were fetched |
| Form URL not working | Workflow is not activated | Toggle the Active switch on |
| "Test URL" works but "Production URL" doesn't | Workflow needs to be active for production URLs | Activate the workflow |
| User gets an old/expired code | The 15-minute window may include older emails | User should request a fresh code from the platform, then submit immediately |
| Name logs to sheet but Timestamp column is empty | n8n cached old column list before Timestamp was added | Re-select the Document and Sheet dropdowns in the Log Request to Sheet node |
| Nothing logs to sheet at all | Operation set to "Append or Update" without a match column | Change operation to **Append Row** |
| Response page shows plain text instead of styled HTML | Content-Type header not set | Add `Content-Type: text/html` header in the Respond to User node options |
| "Code doesn't return items properly" error | Code node returning plain object instead of array | Ensure all Code nodes return `[{ json: { ... } }]` format |

---

## Security Notes

- The form only asks for a name — no passwords or sensitive data
- Codes are displayed once on the form page and not stored anywhere beyond the execution log
- Gmail search is limited to the last 15 minutes, reducing the risk of returning stale data
- Consider clearing n8n execution logs periodically as they may contain extracted codes
- Only share the form URL with authorized users

---

## Customization

### Change the search time window
In the **Gmail** node, change `newer_than:15m` to your preferred window (e.g., `newer_than:5m` for tighter security, `newer_than:30m` for slower email delivery).

### Add more search keywords
In the **Gmail** node's query filter, add additional `OR "keyword"` entries to match emails from specific platforms.

### Change the response format
In the **Extract 2FA Code** node, modify the `message` variable in the JavaScript code to customize what users see. The message is built as HTML so standard HTML tags and inline styles apply.

### Host on a custom domain via Netlify
Create a `_redirects` file in a Netlify site folder with:
```
/get-code    https://your-n8n-domain.com/form/your-form-id    200
```
Deploy the folder to Netlify. Users visit `your-site.netlify.app/get-code` and the n8n form is served transparently under your domain.

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | March 2026 | Initial release — form trigger, Gmail search, code extraction, form response |
| 1.1 | March 2026 | Added Prepare Log Data node for reliable timestamp logging; fixed Code node return format for n8n Cloud compatibility; changed Sheets operation to Append Row; added HTML response styling |
