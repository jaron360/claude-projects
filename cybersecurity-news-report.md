  Build me a daily cybersecurity news brief that runs on my Mac every morning and emails me a nicely designed HTML report through Outlook. Build it end to end, test it, and schedule it.

  ## Environment facts (already known — don't re-discover)
  - macOS, zsh. Claude Code CLI is authenticated via Bedrock, so the **WebSearch tool is NOT available** in headless mode; **WebFetch works**. Don't design around WebSearch.
  - I use the **new Outlook for Mac**. Send email via AppleScript (no SMTP or credentials).
  - Recipient: <YOUR_EMAIL>
  - `claude` is on PATH only via my .zshrc, so a launchd job won't find it. Set PATH explicitly in the script (include the directory from `which claude`, plus /opt/homebrew/bin, /usr/local/bin and the system dirs).

  ## Architecture
  Project dir: `~/cyber-brief/`. Scheduler: a launchd LaunchAgent (not GitHub Actions, and not Claude Code's /loop or cron tools, which only run while a session is open).

  Flow: launchd → `run.sh` → `fetch_feeds.py` collects recent headlines → headless `claude -p` uses the project skill to write the HTML → `email_body.py` cleans the HTML for Outlook → `send_outlook.applescript` sends it.

  ## Files to create

  ### 1. `fetch_feeds.py` (Python stdlib only)
  - Takes a lookback window in hours (default 24). Fetches these RSS/Atom feeds with a browser User-Agent and a 30s timeout:
    BleepingComputer https://www.bleepingcomputer.com/feed/,
    The Hacker News https://feeds.feedburner.com/TheHackersNews,
    SecurityWeek https://www.securityweek.com/feed/,
    Krebs https://krebsonsecurity.com/feed/,
    Dark Reading https://www.darkreading.com/rss.xml,
    The Record https://therecord.media/feed,
    CISA Advisories https://www.cisa.gov/cybersecurity-advisories/all.xml,
    CISA News https://www.cisa.gov/news.xml
  - Parse both RSS `<item>` and Atom `<entry>`. Handle RFC 822 and ISO dates.
  - Print a section per source (`## Source`) and only items inside the window: `- [YYYY-MM-DD HH:MMZ] Title`, then the link, then a summary of up to 300 characters. Strip HTML from summaries with regex + `html.unescape`; don't parse them as 
  XML.
  - If one feed fails, print `FEED ERROR` for it and keep going.

  ### 2. Skill: `.claude/skills/cyber-brief/SKILL.md` + `template.html`
  SKILL.md instructions:
  - Input: the candidate headlines plus the titles already reported in recent briefs.
  - Pick the **top 3–5 events**. Rank by: active exploitation, zero-days or CISA KEV additions first; then breadth of impact; then severity (critical CVSS, RCE, auth bypass, supply chain). Merge duplicate coverage of the same story. Skip 
  funding news, product launches, opinion pieces and non-security tech news. Skip already-reported events unless there's a major new development.
  - Use **WebFetch** on each selected article (and the vendor advisory or CVE page if linked) to get the details.
  - Each event gets these sections: **What happened** (2–3 sentences); **How the attack worked** in plain language a non-specialist can follow, with analogies welcome. Use only details stated in sources; if none are published, say so 
  explicitly and never guess. Then **Impact**, **Mitigation** (patch versions, config changes, IOCs, workarounds, or "none published"), **Severity** (Critical/High/Medium) and **Sources**.
  - Add an optional "Also noteworthy" list: up to 5 one-liners with links.
  - Keep all CSS inline and escape `<`, `>` and `&` in text.
  - **Output must be ONLY the HTML document**, from `<!DOCTYPE html>` to `</html>`, with no markdown fences or commentary. Put this comment right after the doctype: `<!-- REPORTED: title one || title two || ... -->`. It's used to avoid 
  repeating stories.

  template.html, an email-safe design **styled after starlink.com**:
  - Table-based layout, max width 680px, pure black background (set both `bgcolor="#000000"` and inline `background`), `<meta name="color-scheme" content="dark">`.
  - Font stack: `'D-DIN','DIN Alternate','DIN Next','Helvetica Neue',Helvetica,Arial,sans-serif`.
  - White text `#ffffff` for primary content, `#e6e6e6` for body text, `#7d7d7d` for labels, `#b3b3b3` for secondary text. Thin `#2b2b2b` dividers. No rounded corners.
  - Header: a small uppercase label "DAILY CYBERSECURITY BRIEF" (letter-spacing 4px, gray), a large uppercase date (34px), then a one-sentence overview.
  - "At a glance" list: zero-padded numbers (01, 02…) with an uppercase bold title and a one-liner for each event.
  - Event block, repeated per event:
    - The number, plus an *outlined* severity badge (1px border and text in the severity color, uppercase, letter-spaced). Colors: Critical `#ff4d4d`, High `#ff9f43`, Medium `#ffd166`. These are the only accent colors.
    - The title in uppercase at 24px.
    - Section labels: 11px uppercase, letter-spacing 3px, gray.
    - "How the attack worked" in a `#111111` panel with a 2px white left border.
    - "Mitigation" in a `#111111` panel with a 1px `#2b2b2b` border.
    - Sources as white outlined uppercase button-style links ("OUTLET →").
  - Footer: small uppercase gray disclaimer: "Generated by Claude from public security news feeds · Verify with linked sources before acting".
  - Use `{{PLACEHOLDER}}` markers for every field. SKILL.md should tell Claude to Read the template and fill it in.

  ### 3. `email_body.py`
  New Outlook only renders an email as HTML if the body **starts at `<html>`**. A doctype, comments or `<head>` before it make the email show up as raw code. This script reads the report, removes all `<!-- -->` comments, drops everything
  before `<html`, removes `<head>…</head>`, and prints the result.

  ### 4. `send_outlook.applescript`
  Usage: `osascript send_outlook.applescript <html-file> <subject> <recipient>`. Read the file as UTF-8 (`«class utf8»`). Then `make new outgoing message with properties {subject:…, content:…, has html:true}` — **`has html:true` is 
  required** or new Outlook sends plain text. Add a to-recipient and `send`.

  ### 5. `run.sh` (`#!/bin/zsh -l`, `set -euo pipefail`)
  - Export PATH explicitly (see above), set `RECIPIENT`, and `cd` to the script's own directory. Create `reports/` and `logs/`.
  - Flags: `--force` (ignore the time and already-sent guards), `--no-send` (generate only).
  - Guards: unless forced, exit if it's before 08:00, and exit if `reports/$TODAY.sent` exists.
  - Lookback window: 24h, or 72h on Mondays to cover the weekend.
  - Repeat check: collect the `REPORTED:` comments from the last 3 *dated* reports only. Use the zsh glob `reports/<->-<->-<->.html(N)` so it doesn't match `*.email.html`, and exclude today's report.
  - Build a prompt: "Use the cyber-brief skill to write today's brief. Today is <long date>." followed by the candidate headlines and the already-reported list. Pipe it on stdin into `claude -p --allowedTools "Skill,Read,WebFetch"`, sending
  stderr to `logs/claude-stderr.log`.
  - Extract only the `<!DOCTYPE html>` … `</html>` range into `reports/$TODAY.html`. If there's no `</html>`, save the raw output to `logs/$TODAY-raw.txt` and exit with an error.
  - To send: `python3 email_body.py` → `reports/$TODAY.email.html`, then run the AppleScript with the subject `Cybersecurity Brief — <Mon D, YYYY>`. After that, touch the `.sent` marker and log a line.

  ### 6. LaunchAgent `~/Library/LaunchAgents/com.<my-username>.cyber-brief.plist`
  - ProgramArguments: `/bin/zsh -l ~/cyber-brief/run.sh` (use absolute paths).
  - StartCalendarInterval at 8:00 (launchd runs a missed job when the Mac wakes).
  - RunAtLoad true, to cover days the Mac was off at 8. run.sh's guards prevent duplicate or early sends.
  - Send stdout and stderr to `~/cyber-brief/logs/launchd.log`.
  - Lint it with `plutil -lint` and load it with `launchctl bootstrap gui/$(id -u) <plist>`.

  ## Testing (do all of these)
  1. Run `python3 fetch_feeds.py 24` and confirm headlines come back.
  2. Run `./run.sh --force --no-send` and confirm `reports/<today>.html` is created and contains the REPORTED comment.
  3. Simulate launchd's bare environment: `env -i HOME=$HOME USER=$USER /bin/zsh -l ./run.sh --force --no-send`. It must find `claude`; exit 127 means PATH is wrong.
  4. Send one real email to me and ask me to confirm it renders as formatted HTML, not raw code.
  5. Touch today's `.sent` marker before loading the LaunchAgent so it doesn't send again right away, then check `logs/launchd.log` shows "already sent today; skipping".

  Tell me that the first time it runs from launchd, macOS may ask permission to let zsh/osascript control Microsoft Outlook, and that I need to click Allow. Finish with a short summary of the file paths and these commands: send now (`run.sh
  --force`), generate without sending (`--no-send`), and disable (`launchctl bootout gui/$(id -u)/com.<my-username>.cyber-brief`).
