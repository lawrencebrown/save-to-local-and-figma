# Screenshot Drop — Setup Guide

This tool lets you take a full-page screenshot of any webpage and send it straight into a Figma file with one click. It has three parts that need to be set up once:

1. **Cloudflare Worker** — a free relay that temporarily holds your screenshots
2. **Chrome Extension** — the button you click in your browser
3. **Figma Plugin** — the thing that pulls the screenshot into Figma

Follow each section in order. The whole setup takes about 15 minutes.

---

## Part 1 — Cloudflare Worker

The Worker is a tiny free server that sits between your browser and Figma. Screenshots are held there for up to 24 hours.

### Step 1.1 — Create a Cloudflare account

Go to **cloudflare.com** and sign up for a free account if you don't have one. You don't need to add a domain or pay anything.

### Step 1.2 — Create the Worker

1. Once logged in, click **Workers & Pages** in the left sidebar
2. Click **Create** → **Create Worker**
3. Give it any name (e.g. `screenshot-drop`) and click **Deploy**
4. You'll land on a page showing your new worker. Click **Edit Code**
5. You'll see a code editor with a "Hello World" example. **Select all the code and delete it.**
6. Open the file `figma-plugin/worker.js` from this project folder and copy its entire contents
7. Paste it into the Cloudflare code editor
8. Click **Deploy** (top right)

### Step 1.3 — Create the KV storage

Your Worker needs a place to store screenshots temporarily. That's KV (Key-Value storage).

1. In the Cloudflare sidebar, click **Workers & Pages** → **KV**
2. Click **Create a namespace**
3. Name it `SCREENSHOTS` and click **Add**

Now connect it to your Worker:

1. Go back to **Workers & Pages** → click on your worker (e.g. `screenshot-drop`)
2. Click the **Settings** tab
3. Scroll down to **Bindings** → click **Add**
4. Choose **KV Namespace**
5. Set **Variable name** to `SCREENSHOTS` (must be exactly this)
6. Select the `SCREENSHOTS` namespace you just created
7. Click **Save**

### Step 1.4 — Add a secret key

This is a password that stops anyone else from using your Worker.

1. Still on your Worker's Settings tab, scroll to **Environment Variables**
2. Click **Add variable**
3. Set the name to `SECRET_KEY`
4. Set the value to any password you choose — write it down, you'll need it later (e.g. `my-secret-123`)
5. Click **Encrypt** to keep it secure
6. Click **Save and deploy**

### Step 1.5 — Copy your Worker URL

1. Click the **Overview** tab of your Worker
2. You'll see a URL like `https://screenshot-drop.yourname.workers.dev`
3. Copy this — you'll need it in Part 2 and Part 3

---

## Part 2 — Figma Setup

You need two things from Figma: a Personal Access Token (PAT) and your Team ID.

### Step 2.1 — Get a Figma Personal Access Token

This lets the extension browse your Figma files.

1. Open **figma.com** and log in
2. Click your profile picture (top left) → **Settings**
3. Scroll down to **Personal access tokens**
4. Click **Generate new token**
5. Give it a name like `Screenshot Drop` and click **Generate token**
6. **Copy the token immediately** — Figma won't show it again. It starts with `figd_`

### Step 2.2 — Get your Figma Team ID

1. In Figma, click on your team name in the left sidebar to go to the team page
2. Look at the URL in your browser — it will look like:
   `https://www.figma.com/files/team/123456789012345678/Your-Team`
3. The long number in the URL is your Team ID — copy it

### Step 2.3 — Install the Figma Plugin

1. In Figma, open any file
2. Click the Figma logo (top left menu) → **Plugins** → **Development** → **Import plugin from manifest**
3. Click **Choose a manifest.json file**
4. Navigate to this project folder → `figma-plugin` → select `manifest.json`
5. Click **Open**

The plugin "Screenshot Drop" is now installed as a development plugin.

### Step 2.4 — Configure the Figma Plugin

1. In any Figma file, open the plugin: **Plugins** → **Development** → **Screenshot Drop**
2. You'll see two fields: **Worker URL** and **Worker Secret Key**
3. Enter your Worker URL from Step 1.5 (e.g. `https://screenshot-drop.yourname.workers.dev`)
4. Enter your secret key from Step 1.4 (e.g. `my-secret-123`)
5. Click **Save**

---

## Part 3 — Chrome Extension

### Step 3.1 — Load the extension

Chrome doesn't install this from the store — you load it directly from the project folder.

1. Open Chrome and go to `chrome://extensions` in the address bar
2. Turn on **Developer mode** (toggle in the top right corner)
3. Click **Load unpacked**
4. Navigate to this project folder (the one containing `manifest.json`, `popup.html`, `background.js`, etc.) — **not** the `figma-plugin` subfolder, the main folder
5. Click **Select** (or **Open**)
6. You should see "Screenshot Tab" appear in your extensions list

You'll now see the extension icon in your browser toolbar. If you don't see it, click the puzzle piece icon in Chrome's toolbar and pin it.

### Step 3.2 — Configure the extension

1. Click the extension icon in your toolbar
2. A setup form appears asking for four things:

   **Figma Personal Access Token** — paste the token from Step 2.1 (`figd_...`)

   **Figma Team URL or ID** — paste either the full Figma team URL or just the numeric ID from Step 2.2

   **Cloudflare Worker URL** — paste your Worker URL from Step 1.5

   **Worker Secret Key** — paste your secret key from Step 1.4

3. Click **Save**

The extension will load your Figma projects and files. Setup is complete.

---

## How to Use It

### Taking a screenshot

1. Go to any webpage you want to capture
2. Click the **Screenshot Tab** extension icon
3. A popup appears showing your Figma projects and files
4. **Projects tab**: select a Project from the first dropdown, then select a File from the second
5. **Send to draft tab**: if your file isn't in a project, paste a Figma file URL directly, or select a previously saved draft from the list
6. Click **Send It →**
7. The popup shows a spinner while it captures and uploads (may take a few seconds for long pages)
8. On success you'll see a confirmation with an "Open in Figma →" link

The screenshot is copied to your clipboard automatically. You can also click **Save to disk** to download the PNG file.

### Placing the screenshot in Figma

1. In Figma, navigate to the page where you want the screenshot
2. Open the plugin: **Plugins** → **Development** → **Screenshot Drop**
3. The plugin shows a list of pending screenshots
4. Click a screenshot to select it, then click **Place Selected**
5. The screenshot appears as a frame on your current Figma page

### Multiple screenshots

You can take several screenshots before opening Figma — they all queue up. The plugin shows the full list and you pick which ones to place. Screenshots expire after 24 hours.

### Sending again

If you accidentally deleted a screenshot in Figma, click **Send Again** in the extension popup. It re-sends the last screenshot without recapturing.

---

## Troubleshooting

**"Failed to load projects"** — Your Figma token may be expired or incorrect. Click the gear icon in the extension to go back to settings and re-enter it. Make sure it starts with `figd_`.

**"Worker 401"** — The secret key doesn't match. Double-check it's the same in the extension settings and the Cloudflare Worker environment variable.

**"Worker 500" or fetch error** — The Worker may have an error. Go to your Cloudflare dashboard → your Worker → **Logs** to see what went wrong. Most common cause: the KV binding isn't set up correctly (re-check Step 1.3, the variable name must be exactly `SCREENSHOTS`).

**Extension shows "Capturing…" forever** — The page may have blocked the Chrome Debugger. Reload the page and try again. This sometimes happens on certain browser-internal pages (`chrome://`, `about:`, PDF files) — the extension can't capture those.

**Screenshot looks correct in the extension but is cut off in Figma** — The Worker upload probably timed out. Click **Send Again** in the extension popup.

**Screenshot has scrollbars visible** — Some pages use custom JavaScript-powered scrollbars that are harder to hide. This is a known limitation on certain sites.

**Plugin shows "No screenshots pending"** — Either no screenshot has been taken yet, or they expired (24-hour limit). Take a new screenshot from the extension.

---

## Updating the Worker

When the `figma-plugin/worker.js` file changes:

1. Go to Cloudflare → Workers & Pages → your worker → **Edit Code**
2. Replace all code with the new version
3. Click **Deploy**

Your KV namespace and secret key are not affected — they stay the same.

## Reloading the Extension After Changes

When any extension file changes:

1. Go to `chrome://extensions`
2. Find "Screenshot Tab" and click the **reload** icon (circular arrow)
3. Close and reopen the popup

## Reloading the Figma Plugin After Changes

When `code.js` or `ui.html` changes:

1. In Figma, close the plugin if it's open
2. Open it again — Figma automatically picks up the latest saved files from disk
