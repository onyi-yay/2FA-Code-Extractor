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
User fills form  -->  Request logged  -->  Gmail searched  -->  Code extracted  -->  Code displayed
```

1. **User opens the form link** and enters their name
2. **Request is logged** to a Google Sheet (audit trail)
3. **Gmail is searched** for unread 2FA emails from the last 15 minutes
4. **The code is extracted** using pattern matching (supports 4-8 digit/alphanumeric codes)
5. **The code is displayed** directly on the form response page

The entire process takes a few seconds.

---

## Workflow Nodes

| # | Node | Purpose |
|---|------|---------|
| 1 | **Form Trigger** | Web form where users enter their name |
| 2 | **Log Request to Sheet** | Logs the request to Google Sheets for tracking |
| 3 | **Gmail - Fetch Recent 2FA Emails** | Searches Gmail for unread verification emails (last 15 min) |
| 4 | **Extract 2FA Code** | Parses the email body/subject to find the code, builds the response |
| 5 | **Respond to User** | Displays the result to the user on the form page |

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

2. **Configure Google Sheets node**
   - Open the **Log Request to Sheet** node
   - Select your Google Sheet from the **Document** dropdown (or create a new one)
   - Select the sheet tab from the **Sheet** dropdown
   - Make sure the sheet has a column header: `Full Name`

3. **Verify Gmail credentials**
   - Open the **Gmail - Fetch Recent 2FA Emails** node
   - Confirm your Gmail OAuth2 credential is connected
   - This must be the Gmail account that receives the 2FA codes

4. **Test the workflow**
   - Click the **Form Trigger** node and copy the **Test URL**
   - Open the URL in a browser, enter a name, and submit
   - Trigger a real 2FA code from any platform, then submit the form again to verify extraction

5. **Activate for production**
   - Toggle the **Active** switch in the top-right corner
   - Copy the **Production URL** from the Form Trigger node
   - Share this URL with your users

---

## What Users See

### When a code is found:
> Hi [Name]!
>
> =============================
>    YOUR CODE:  482910
> =============================
>
> Go enter this code now — it expires in a few minutes.

### When no code is found:
> Hi [Name],
>
> We couldn't find a verification code just yet.
>
> This usually happens when:
>   1. The email is still on its way — wait about 30 seconds and try again
>   2. The code already expired — go back to the platform, request a new code, then submit this form immediately

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

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| "Code not found" every time | Gmail credentials may not be for the shared email | Verify the Gmail node is connected to the correct account |
| Code returned is wrong/random | Another email matched the search keywords | Check the Gmail node output to see which emails were fetched |
| Form URL not working | Workflow is not activated | Toggle the Active switch on |
| "Test URL" works but "Production URL" doesn't | Workflow needs to be active for production URLs | Activate the workflow |
| User gets an old/expired code | The 15-minute window may include older emails | User should request a fresh code from the platform, then submit immediately |

---

## Audit & Logging

Every request is logged to Google Sheets with the user's name and timestamp. This helps you:
- Track who is requesting codes and when
- Identify if someone is submitting excessive requests
- Maintain a record for accountability

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
In the **Extract 2FA Code** node, modify the `message` variable in the JavaScript code to customize what users see.

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | March 2026 | Initial release — form trigger, Gmail search, code extraction, form response |
