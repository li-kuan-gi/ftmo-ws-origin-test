# FTMO WebSocket Origin Test

A static, browser-only diagnostic page for checking whether the FTMO MT5 Web Terminal WebSocket endpoint accepts a connection initiated from the deployed GitHub Pages origin.

## What it does

- Uses the browser's native WebSocket API.
- Defaults to `wss://mt5demowebterminal.ftmo.com/terminal`, the FTMO terminal endpoint used in the preceding working Python test. Change it only if your FTMO Client Area provides a different authorized endpoint.
- Sends no MT5 credentials and no application-level protocol frames.
- Reports whether the browser receives the WebSocket `open` event.
- Logs timestamps in UTC and message sizes only; it does not print WebSocket payload contents.

An `open` event proves only that the WebSocket opening handshake completed for this page's Origin. It does not prove that the MT5 bootstrap, encryption, login, or historical-bar request will work from the browser.

A failed connection is intentionally reported as inconclusive: browser JavaScript does not expose the handshake's HTTP rejection status or detailed network failure reason.

## Publish with GitHub Pages

1. Create a new GitHub repository, for example `ftmo-ws-origin-test`.
2. Upload `index.html` to the repository root. `README.md` is optional for the site.
3. Open **Settings → Pages**.
4. Under **Build and deployment**, choose **Deploy from a branch**.
5. Choose branch `main` and folder `/(root)`, then save.
6. Wait for the Pages deployment to finish. Open the site URL shown in Settings → Pages. It will usually look like `https://YOUR-USERNAME.github.io/ftmo-ws-origin-test/`.
7. Run the test from that HTTPS URL, not by opening the file locally.
8. Click **複製診斷紀錄** and share the resulting sanitized report. Do not share passwords, cookies, tokens, or authenticated frames.

GitHub's official Pages publishing guide: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site

## Important

This is a diagnostic-only page. It does not implement the MT5 binary protocol, encryption bootstrap, login, or M15 history download. Do not enter your trading credentials into this page.
