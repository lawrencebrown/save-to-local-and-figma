# Screenshot Drop

A personal-use tool that captures a full-page screenshot of any webpage and places it directly into a Figma file — one click in the browser, one click in Figma.

Three parts work together:

- **Chrome Extension** — click the icon, pick a destination Figma file, click Send. Captures the full scrollable page and uploads it.
- **Cloudflare Worker** — a free relay that holds screenshots in a queue for up to 24 hours.
- **Figma Plugin** — opens inside Figma, shows the queue, place a screenshot onto the current page with one click.

---

## Build it with an AI agent

The fastest way to get your own copy running is to hand the spec to an AI coding agent (Claude Code, Cursor, etc.) and let it build the whole thing.

1. Give the agent the file `AGENT_SPEC.md` from this repo
2. Tell it: *"Build this project from scratch following the spec exactly"*
3. Follow `SETUP_GUIDE.md` to deploy the Cloudflare Worker, load the Chrome extension, and install the Figma plugin

The spec is detailed enough that a capable agent can produce a working build in one shot.

---

## Build it yourself

If you'd rather do it manually, `SETUP_GUIDE.md` walks through the full setup step by step — no prior experience needed. It covers creating the Cloudflare Worker, getting your Figma credentials, loading the extension in Chrome, and installing the plugin.

---

## How it works

1. Navigate to any webpage and click the extension icon
2. Select your target Figma file from the dropdown (or paste a Figma URL)
3. Click **Send It →** — the extension scrolls the full page, stitches the segments into a single PNG, and uploads it to the Worker
4. In Figma, open the Screenshot Drop plugin, select the screenshot from the queue, click **Place Selected**
5. The screenshot appears as a frame on your current Figma page

The PNG is also saved to your downloads folder and copied to clipboard on every capture.
