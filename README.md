# Puwesto

An office seating planner built as a Google Apps Script web app, with a Google Sheet as its data store.

## Architecture

```
Browser (Index.html: HTML, CSS, JS)
        │  google.script.run
        ▼
Apps Script (Code.gs)
        │  SpreadsheetApp
        ▼
Google Sheet (People, Plan tabs)
```

- The client is a single `Index.html` with all CSS and JavaScript inline, served by `doGet()`.
- The client calls server functions through `google.script.run`. There is no separate API or database.
- All storage calls go through one storage module. On Apps Script it reads and writes the Sheet; locally it swaps in in-memory sample data, so the app runs without Google.
- Writes are debounced on the client before being sent to the server.

## Tech stack

| Layer | Tool |
| --- | --- |
| Frontend | Plain HTML, CSS, and JavaScript (no framework, no build step) |
| Backend | Google Apps Script (V8 runtime) |
| Storage | Google Sheets |
| Deployment | GitHub Actions + [clasp](https://github.com/google/clasp) |
| Custom links | GitHub Pages redirects |
| Tests | Node built-in test runner |

## Project structure

```
puwesto/
├── src/
│   ├── Code.gs            # doGet(), getData(), savePlan(), savePeople()
│   ├── Index.html         # Client app: markup, styles, scripts
│   └── appsscript.json    # Manifest: runtime, web app access settings
├── docs/
│   ├── index.html         # Redirect to production web app
│   └── staging/
│       └── index.html     # Redirect to staging web app
├── .github/
│   └── workflows/         # Deploy workflows
├── .clasp.json            # clasp project config (rootDir: src)
└── README.md
```

## Local development

**Requirements:** VS Code, Node, Git. No Google login, server, or database needed.

```bash
git clone https://github.com/jjepss/puwesto.git
cd puwesto
```

- Open `src/Index.html` in a browser, or use the VS Code **Live Server** extension for reload on save.
- Locally, the storage module uses in-memory sample data instead of the Sheet.
- Run tests:
  ```bash
  node --test
  ```

## Environments

| | Staging | Production |
| --- | --- | --- |
| Apps Script project | Puwesto Staging | Puwesto |
| Google Sheet | Test Sheet with sample data | Live Sheet |
| Branch | `develop` | `main` |
| Link | `jjepss.github.io/puwesto/staging` | `jjepss.github.io/puwesto` |

Staging renders a `STAGING` badge in the header.

## Branching

```
feature/* ──PR──▶ develop ──PR (review required)──▶ main
                     │                                │
                  staging                        production
```

1. Branch from `develop`.
2. PR into `develop`; merge deploys to staging.
3. Test on staging.
4. PR `develop` into `main`; requires review, merge deploys to production.

## CI/CD

On push to `develop` or `main`, the workflow:

1. Installs clasp
2. Writes `~/.clasprc.json` from a repo secret
3. Runs `clasp push` to the environment's script ID
4. Runs `clasp deploy --deploymentId <id>` to update the existing deployment in place, so the web app URL never changes

**Repo secrets**

| Secret | Value |
| --- | --- |
| `CLASPRC_JSON` | Contents of `~/.clasprc.json` |
| `STAGING_SCRIPT_ID` | Staging Apps Script project ID |
| `STAGING_DEPLOYMENT_ID` | Staging web app deployment ID |
| `PROD_SCRIPT_ID` | Production Apps Script project ID |
| `PROD_DEPLOYMENT_ID` | Production web app deployment ID |

## One-time setup

1. Turn on the Apps Script API at [script.google.com/home/usersettings](https://script.google.com/home/usersettings).
2. `npm install -g @google/clasp` and `clasp login`.
3. Create both Apps Script projects and deploy each once manually to get its deployment ID.
4. Set `SHEET_ID` in each project's Script Properties.
5. Add the repo secrets above.
6. Turn on GitHub Pages from `main`, `/docs`, and set each redirect page to its web app URL.

## Access

Set in `appsscript.json` and deployed with the code:

```json
"webapp": {
  "executeAs": "USER_ACCESSING",
  "access": "ANYONE"
}
```

- The app runs as the signed-in user, so Sheet permissions control access. Sharing or unsharing the Sheet adds or removes users.
- `ANYONE` here means anyone with a Google account; without Sheet access, `getData()` fails.
- Users authorize the script on first open and may see an "unverified app" warning.
- GitHub Pages uses redirects, not iframes, because Google sign-in often breaks inside embedded Apps Script apps.