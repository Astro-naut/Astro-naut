# ShelfTag — Local Setup Guide

> **Why local n8n?**  
> n8n Cloud routes webhooks through Cloudflare which strips response bodies (Content-Length: 0 bug).  
> Running n8n on your own machine eliminates that layer entirely — your HTML talks directly to `localhost:5678`.

---

## What you'll build

```
shelftag-scanner.html          ←  runs on localhost:8080 (Python HTTP server)
        │
        ├─ Google Books API     ←  direct fetch for book data (no n8n needed)
        │
        └─ localhost:5678       ←  local n8n webhook (fire-and-forget)
                │
                └─ Google Sheets  ←  your personal library database
```

---

## Step 1 — Install Node.js (if not already installed)

n8n requires **Node.js 18 or 20**.

1. Go to <https://nodejs.org> and download the **LTS** version.  
2. Run the installer and tick "Add to PATH".  
3. Open a new terminal and confirm:
   ```
   node --version    # should show v18.x or v20.x
   npm --version
   ```

---

## Step 2 — Install n8n locally

```bash
npm install -g n8n
```

This installs n8n as a global command. It takes 1–3 minutes.

### Optional but recommended — set a data folder

By default n8n stores data in `~/.n8n`. You can point it somewhere specific:

```bash
# Windows (PowerShell)
$env:N8N_USER_FOLDER = "C:\Users\YourUsername\Desktop\shelftag\.n8n"

# Mac / Linux
export N8N_USER_FOLDER="$HOME/Desktop/shelftag/.n8n"
```

---

## Step 3 — Start n8n

```bash
n8n start
```

You should see output like:
```
Editor is now accessible via:
http://localhost:5678/
```

Open <http://localhost:5678> in your browser. Create your owner account on first run (name + email + password — these are **local only**, not sent anywhere).

---

## Step 4 — Set up Google Sheets credential in n8n

### 4a. Create a Google Cloud project & enable APIs

1. Go to <https://console.cloud.google.com/>
2. Create a new project (e.g. "ShelfTag")
3. Go to **APIs & Services → Library** and enable:
   - **Google Sheets API**
   - **Google Drive API** (needed to list spreadsheets)

### 4b. Create OAuth 2.0 credentials

1. Go to **APIs & Services → Credentials → Create Credentials → OAuth client ID**
2. Application type: **Web application**
3. Name: `n8n ShelfTag`
4. Under **Authorised redirect URIs**, add:
   ```
   http://localhost:5678/rest/oauth2-credential/callback
   ```
5. Click **Create** — copy the **Client ID** and **Client Secret**

### 4c. Add the credential in n8n

1. In n8n: **Credentials → New → Google Sheets OAuth2 API**
2. Paste your Client ID and Client Secret
3. Click **Sign in with Google** — authorise in the popup
4. Name it: `Google Sheets — ShelfTag`
5. Save ✓

---

## Step 5 — Set up your Google Sheet

1. Create a new Google Sheet at <https://sheets.google.com>
2. Name the sheet tab (the tab at the bottom): **`Library`** (exact spelling)
3. In **row 1**, add these column headers exactly:

   | A | B | C | D | E | F | G | H | I | J | K | L | M |
   |---|---|---|---|---|---|---|---|---|---|---|---|---|
   | ISBN | Title | Author | Publisher | Year | Pages | Categories | Language | Description | Cover URL | Date Added | My Rating | Notes |

4. Copy the **Spreadsheet ID** from the URL:
   ```
   https://docs.google.com/spreadsheets/d/  ← YOUR_ID_IS_HERE  /edit
   ```

---

## Step 6 — Import the n8n workflow

1. In n8n: top-left menu → **Import from file**
2. Select `shelftag-n8n-workflow.json` from this folder
3. The workflow opens with 5 nodes:
   ```
   Webhook → Check Duplicate ISBN → Is Duplicate? → [true]  Already in Library (no-op)
                                                  → [false] Add Book to Sheet
   ```

### Configure the two Google Sheets nodes

Click **"Check Duplicate ISBN"** node:
- **Document**: change `YOUR_SPREADSHEET_ID_HERE` to your actual Spreadsheet ID
- **Credential**: select `Google Sheets — ShelfTag`
- Keep Sheet Name as `Library`
- Save the node

Click **"Add Book to Sheet"** node:
- **Document**: same Spreadsheet ID
- **Credential**: same credential
- Save the node

### Activate the workflow

Toggle the **Active** switch at the top-right of the canvas to **ON**.  
The webhook URL is now live at:
```
http://localhost:5678/webhook/shelftag-add-book
```

> **Testing mode:** Before activating, you can use the test URL instead:  
> `http://localhost:5678/webhook-test/shelftag-add-book`  
> Click the **Test workflow** button (▶) then scan a book — the execution shows in the canvas.

---

## Step 7 — Run the HTML frontend

Open a terminal in the `shelftag` folder:

```bash
# Windows
cd C:\Users\YourUsername\Desktop\shelftag

# Mac / Linux
cd ~/Desktop/shelftag
python3 -m http.server 8080
```

Then open: <http://localhost:8080/shelftag-scanner.html>

> ⚠️ **Never open the HTML by double-clicking** — `file://` blocks all network requests.

---

## Step 8 — Test the full flow

1. The **n8n status dot** in the top-right header should turn **green** (connected ✓)
2. Type an ISBN in the input box (try `9780385333348` — The Da Vinci Code)
3. Press **Look up** or Enter
4. Book details appear — cover, title, author, year, etc.
5. Click **+ Add to Library**
6. Check your Google Sheet — a new row should appear within 2–3 seconds

---

## Everyday use (once set up)

Every time you want to use ShelfTag:

1. **Terminal 1** — keep n8n running:
   ```bash
   n8n start
   ```
2. **Terminal 2** — keep Python server running:
   ```bash
   cd C:\Users\YourUsername\Desktop\shelftag
   python -m http.server 8080
   ```
3. Open <http://localhost:8080/shelftag-scanner.html>

Tip: create a simple `.bat` (Windows) or `.sh` (Mac/Linux) script to launch both with one double-click:

**Windows** — `start-shelftag.bat`:
```bat
@echo off
start "n8n" cmd /k "n8n start"
timeout /t 3
start "" "http://localhost:8080/shelftag-scanner.html"
cd /d "%~dp0"
python -m http.server 8080
```

**Mac/Linux** — `start-shelftag.sh`:
```bash
#!/bin/bash
n8n start &
sleep 3
python3 -m http.server 8080 &
open "http://localhost:8080/shelftag-scanner.html"
wait
```

---

## Camera scanning (optional)

The app uses **QuaggaJS** for camera barcode scanning. It works automatically in Chrome/Edge on localhost.

- Click **📷 Start Camera**  
- Hold the book's barcode steady inside the green box  
- It auto-detects ISBN-13 (EAN-13) and triggers a lookup  
- If the camera doesn't work: use manual ISBN entry (just as fast)

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| n8n status dot stays red | Make sure `n8n start` is running; check the webhook URL in the HTML matches |
| "No book found for ISBN" | ISBN might be ISBN-10 — try adding "978" prefix; or try searching on books.google.com manually |
| Book added but not in Sheet | Check the workflow execution log in n8n (Executions menu); verify Spreadsheet ID and sheet tab name are `Library` |
| Google Sheets auth error | Re-authorise the credential in n8n (Credentials → edit → Sign in with Google again) |
| Camera permission denied | Open `chrome://settings/content/camera` and allow `localhost` |
| Python server error "address in use" | Another process is using port 8080; try `python -m http.server 8081` and update the CORS allowedOrigins in n8n webhook node |
| n8n not installed / `n8n` not found | Run `npm install -g n8n` again; restart your terminal |

---

## Architecture notes

- **No cloud dependency** — everything runs on `localhost`
- **Fire-and-forget POST** — the HTML sends the book data and immediately shows success; it does not wait for n8n to finish writing to Google Sheets
- **Duplicate prevention** — if you scan the same ISBN twice, the workflow detects the existing row and routes to the no-op node (skips the append)
- **LocalStorage history** — the "Recently Added" sidebar is stored in your browser; it is separate from the Google Sheet and can be cleared without affecting your library
- **Google Books API** — called directly from the browser with no API key required (rate-limited to ~1000 requests/day per IP; add a key in the HTML config section for higher limits)
