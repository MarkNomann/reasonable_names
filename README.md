# Reasonable Names — Google Drive Photo Auto-Renamer

Submit a folder name through a simple form, and this workflow automatically finds every image inside that Google Drive folder, analyzes each one with a local AI vision model, and renames the file with a clean, descriptive name — then emails you when it's done.

## What This Workflow Does

1. **You fill out a short form** with two fields: the Drive folder name, and an email to notify when finished
2. The workflow **finds that folder** in your Google Drive by name
3. It **lists every file inside it**
4. It **loops through each file one at a time**
5. **Non-image files are skipped automatically** (only files with an `image/...` MIME type are processed)
6. Each image is **downloaded**, sent to a **local AI vision model**, and analyzed
7. The AI's suggested name is **parsed and cleaned up**
8. The file is **renamed directly in Google Drive**
9. Once every file has been processed, you get a **notification email**

## Requirements

- An n8n instance (self-hosted or Cloud) with the **Form Trigger** and **LangChain Ollama** node available
- A **Google account** with access to the Drive folder you want organized
- **Google Drive API credentials** connected in n8n (steps below)
- **Ollama** installed and running, with a vision-capable model pulled (this workflow ships configured for `llava-phi3`, but any vision model works — see "Changing the AI Model" below)
- A **Gmail account** connected in n8n for the completion email (or swap this node for any other email/notification method you prefer)

## Setting Up Google Drive Credentials

This workflow uses **three** Google Drive nodes (Get folder name, get files in folder, Download file, Update file), and they all use the **same credential** — you only need to set this up once.

### Step 1: Create a Google Cloud project and enable the Drive API

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Create a new project (or use an existing one)
3. In the search bar, look up **"Google Drive API"** and click **Enable**

### Step 2: Create an OAuth consent screen

1. In the left menu, go to **APIs & Services → OAuth consent screen**
2. Choose **External** (unless you have a Google Workspace org and want Internal)
3. Fill in the required app name, support email, and developer contact — these are just labels, not functional settings
4. Add your own Google account under **Test users** if the app is in "Testing" mode (this avoids needing Google's full verification process for personal use)

### Step 3: Create OAuth credentials

1. Go to **APIs & Services → Credentials → Create Credentials → OAuth client ID**
2. Application type: **Web application**
3. Under **Authorized redirect URIs**, add the redirect URI n8n gives you — you'll find this exact URL inside n8n when creating the credential in the next step (it looks like `https://your-n8n-instance/rest/oauth2-credential/callback`)
4. Save, then copy the generated **Client ID** and **Client Secret**

### Step 4: Connect the credential in n8n

1. In n8n, go to **Credentials → New → Google Drive OAuth2 API**
2. Paste in the **Client ID** and **Client Secret** from Step 3
3. Click **Connect my account** and sign in with the Google account whose Drive you want to use
4. Save the credential

### Step 5: Assign the credential to each node

Open each of these four nodes in the workflow and make sure the same Google Drive credential is selected:
- **Get folder name**
- **get files in folder**
- **Download file**
- **Update file**

If you import this workflow fresh, n8n will usually prompt you to select a credential for each Google node automatically on first open.

## Setting Up Ollama

1. Install Ollama from [ollama.com](https://ollama.com) if you haven't already
2. Pull a vision-capable model, for example:
   ```
   ollama pull llava-phi3
   ```
3. Make sure Ollama is running (`ollama serve`, or it may already run automatically in the background depending on your install)
4. In n8n, create an **Ollama** credential (Credentials → New → Ollama API) pointing to your Ollama instance (default: `http://localhost:11434`)
5. Assign that credential to the **Analyze image** node

## Setting Up the Notification Email

The **Finish notification** node uses Gmail. Connect a Gmail OAuth2 credential in n8n the same way as any other Google service, and assign it to that node. If you'd rather use a different email provider, swap this node out for n8n's generic **Send Email (SMTP)** node instead — everything else in the workflow stays the same.

## How to Use It

1. Open the **On form submission** node and copy its **Production URL**
2. Visit that URL in a browser
3. Enter the exact name of the Google Drive folder you want processed, and the email address to notify when it's done
4. Submit the form
5. The workflow runs in the background. Check n8n's **Executions** tab to monitor progress
6. You'll receive an email once every file has been checked and renamed

## Changing the AI Model or Provider — No Problem

This workflow is intentionally built so the AI step is a **swappable component**, not something wired tightly into the rest of the logic. You're free to change it however suits you:

- **Swap to a different local Ollama model** — just change the model name in the **Analyze image** node's model field to any vision-capable model you have pulled (e.g. `llava:7b`, `llava:13b`, `bakllava`, or newer models as they become available). No other node needs to change.
- **Swap to a cloud AI provider instead** — if you'd rather use OpenAI, Anthropic, or another vision API (for faster or more accurate results at a small per-image cost), replace the **Analyze image** node with the equivalent node for that provider. You will likely need to adjust:
  - The **prompt** slightly to match how you want the response formatted
  - The **Edit Fields** node's parsing logic, since different providers/models format their JSON responses slightly differently (the current parsing uses regex matching to pull out `id` and `name` values, which is deliberately tolerant of messy formatting — but double-check it still matches your new model's typical output style)
- **Everything downstream stays the same** — the Update file, looping, and notification logic don't care which AI produced the name, only that the Edit Fields node hands off a clean `id` and `name` field.

In short: the AI piece is meant to be the most replaceable part of this workflow. Feel free to experiment with different models or providers as they improve, without needing to rebuild anything else.

## Notes and Limitations

- If multiple folders share the exact same name in your Drive, the workflow will use whichever one Google's API returns first — rename duplicates to avoid ambiguity.
- Only files with an `image/...` MIME type are processed; videos and other file types are automatically skipped, not renamed.
- The AI-generated filenames depend on model quality. Smaller/faster local models (like `llava-phi3`) trade some accuracy for speed and lower memory use.
- This workflow authenticates with **your own** connected Google and Ollama credentials — it does not support separate end-users each connecting their own accounts without additional setup work.
- Execution data logging is disabled in this workflow's settings (`saveDataSuccessExecution: none`) to reduce memory usage during large batches. If you need to debug a specific run, you can temporarily re-enable this in Workflow Settings, but remember to turn it back off for large batches.

## Support

This workflow is provided as-is. You're welcome to modify any part of it — model choice, prompt wording, filtering logic, or notification method — to fit your own setup.


