# Media Day Check-In

A check-in page (`index.html`) that reads and writes **directly to your
Google Sheet** through a small Apps Script Web App (`Code.gs`). Your
existing `onEdit` / `syncAndColor()` logic is untouched — the check-in
add-on just calls it after writing, so `Sorted_View` mirrors `Sheet1`
exactly, same as it always has.

## What changed in your Sheet

Nothing, going forward. Your current layout is used as-is:

| Col | Field | Used how |
|---|---|---|
| A | Team | shown + filterable |
| B | Header Name | shown |
| C | Details | shown |
| **D** | **Check In** | **written by this app** — the only column it touches |
| E | RSVP | shown as a "No RSVP" tag when unchecked; never written |
| F | Update | untouched |
| G | Round | shown |
| H | Pax/Total | shown |

## 1. Add the check-in add-on to Apps Script

1. Open your Sheet → **Extensions → Apps Script**.
2. Your existing script is already there. Open `Code.gs` in this folder,
   copy the whole thing, and paste it over what's in the editor — it's
   your original script unchanged, with the check-in functions added
   below it.
3. In `CHECKIN_CONFIG` near the bottom, change:
   ```js
   sharedSecret: "change-me-to-a-long-random-string"
   ```
   to your own long random string (this is a basic write-lock so a
   leaked URL can't be used to toggle check-ins by strangers).

## 2. Deploy it as a Web App

1. **Deploy → New deployment**.
2. Type: **Web app**.
3. Execute as: **Me**.
4. Who has access: **Anyone**.
5. Deploy, and authorize the permissions it asks for (it needs to edit
   the sheet).
6. Copy the URL it gives you — it ends in `/exec`.

If you edit `Code.gs` later, use **Deploy → Manage deployments → Edit
(pencil) → New version** so the same URL picks up your changes.

## 3. Configure the front end

Open `index.html` and edit the `CONFIG` block near the top:

```js
const CONFIG = {
  EVENT_NAME: "Media Day",
  APPS_SCRIPT_URL: "https://script.google.com/macros/s/.../exec",
  SHARED_SECRET: "change-me-to-a-long-random-string",  // same as Code.gs
  NTFY_TOPIC: "your-event-checkin-topic-change-me",     // "" to disable
  NTFY_SERVER: "https://ntfy.sh",
  POLL_MS: 8000
};
```

For push alerts, subscribe to `NTFY_TOPIC` in the [ntfy app](https://ntfy.sh)
or by opening `https://ntfy.sh/<your-topic>` in a browser tab.

## 4. Deploy to GitHub Pages

```bash
git init
git add index.html Code.gs README.md
git commit -m "Media day check-in app"
git branch -M main
git remote add origin https://github.com/<you>/<repo>.git
git push -u origin main
```

Then: **Settings → Pages → Deploy from branch → main → / (root)**.
Live at `https://<you>.github.io/<repo>/`.

## How it works

- **On load / every 8s**: the page calls the Web App (`doGet`), which
  reads Sheet1 fresh and returns every row's Team, Name, Details, RSVP,
  Round, Pax, and Check In status. This is how multiple devices at the
  door stay in sync — they're all just reading the live Sheet.
- **On tap**: the page calls the Web App (`doPost`) with the row number
  and `in`/`out`. It writes `TRUE`/`FALSE` into column D for that row
  only, then calls your existing `syncAndColor()` so `Sorted_View`
  updates immediately — exactly as if you'd ticked the box by hand.
- **ntfy** is a one-way alert only now (not used for syncing state) —
  fired from the browser right after a successful write.

## Notes

- Rows are matched by **row number**, read fresh on every load. Don't
  manually reorder/insert rows in `Sheet1` while people are actively
  checking in — a resort mid-event could point a stale row number at
  the wrong person until the page next refreshes.
- Anyone with the Web App URL *and* the shared secret can write to
  column D. Keep both out of public view (don't commit the real secret
  to a public repo — consider making the repo private, or swap in a
  placeholder before pushing and set the real one only in your local copy).
