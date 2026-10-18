# AGENTS.md

Technical guide for anyone working in this repo, including AI coding agents. README.md has the short overview.

## Project context

- Puwesto is an office seating planner: a Google Apps Script web app backed by Google Sheets.
- It is a static seating plan, not a desk-booking tool: no dates, check-ins, or reservations. One fixed floor, desktop browsers only.
- It is built by two people as a learning project, one lesson at a time. Code must stay simple enough to explain line by line.
- The MVP is seats only. Zoning, configurable rules, column mapping, multiple floors, and editing people inside Puwesto come after the MVP; do not add them unless asked.

## Architecture

```
Browser (Index.html + app scripts)
        │  google.script.run
        ▼
Apps Script (Code.gs)
        │  SpreadsheetApp
        ├──────────────────────────┐
        ▼                          ▼
Database Sheet               People Sheet
(Person, Team, Seat)         (client's source list, read-only)
```

- The page is served by `doGet()` through `HtmlService.createTemplateFromFile` and a small `include()` helper.
- **Database Sheet:** Puwesto's own data, one tab per table. The app reads and writes it.
- **People Sheet:** the client's people list. Read only during Sync; never written to.
- All Google calls live in one storage file. On Apps Script it uses the Sheets; locally it uses in-memory sample data.
- Seating writes autosave to the database Sheet, debounced on the client.
- Export (PNG, PDF, spreadsheet) is built entirely in the browser, so it also works locally.

## Project structure

```
puwesto/
├── app/                   # The page, as it runs locally
│   ├── index.html         # Markup and CSS; loads each .js with a script tag
│   └── *.js               # data.js, rules.js, desk config, storage, sync, etc.
├── src/                   # What clasp pushes to Apps Script
│   ├── Code.js            # Server code (Code.gs in Apps Script)
│   └── appsscript.json    # Manifest: runtime, web app access settings
├── scripts/
│   └── build.js           # Builds app/ into src/ at deploy
├── tests/                 # Rule tests (Node test runner)
├── docs/
│   ├── index.html         # Redirect to production web app
│   └── staging/
│       └── index.html     # Redirect to staging web app
├── .github/workflows/     # deploy-staging.yml, deploy-production.yml
├── .clasp.json            # clasp project config (rootDir: src)
├── AGENTS.md
└── README.md
```

**Build step:** Apps Script can't serve separate script files, so `scripts/build.js` repackages the page at deploy. Each `app/*.js` becomes `src/<name>.html` wrapped in script tags, and `src/Index.html` is generated from `app/index.html` with each script tag swapped for an `include()`. The built files are gitignored.

## Server functions (`src/Code.js`)

| Function | Purpose |
| --- | --- |
| `doGet()` | Serves the page |
| `getData()` | Reads Person, Team, and Seat; returns them with `canEdit` |
| `savePlan()` | Writes Seat; refuses to run for viewers |
| `previewSync()` | Reads the people Sheet and returns the sync summary without applying |
| `applySync()` | Applies the sync to Person and Team; refuses to run for viewers |

## Stack rules

- Plain HTML, CSS, and JavaScript only. No frameworks, bundlers, TypeScript, or npm runtime dependencies.
- The only libraries are for export: html-to-image (PNG), jsPDF (PDF), and ExcelJS (spreadsheet), loaded with script tags from a CDN. Don't swap ExcelJS for SheetJS; its free version can't write fill colors.
- CSS stays inside `app/index.html`. Each piece of JavaScript is its own `.js` file in `app/`, loaded with a normal script tag.
- Server code uses only built-in Apps Script services (`HtmlService`, `SpreadsheetApp`, `PropertiesService`).
- Browser code and Node can't see Apps Script services. Never reference `SpreadsheetApp`, `HtmlService`, or `google.script.run` outside `src/Code.js` and the storage file.

## Code conventions

- **Rules are pure.** Seating functions (`seat`, `move`, `clearSeat`, `resetAll`) take a plan and return a new plan. Never mutate the input; undo and redo depend on this.
- **Core rules:** anyone can sit at any desk, a person sits at one desk at a time, a desk holds at most one person, and a desk shows its occupant's team color (neutral for no team).
- **Undo and redo** cover seating actions only (seat, move, replace, Clear seat, Reset all). Sync and uploading a plan file are not undoable and clear the history.
- **One storage file.** All Google Sheets calls go through it.
- **Sync has two steps.** `readSource` is the only code that knows the people Sheet's layout; it cleans text and returns a standard list (`key`, `name`, `team`, `role`) plus skipped rows with reasons. `mergePeople` is generic and must not depend on the source's columns. When the source changes, change only `readSource`.
- **IDs are Puwesto's own.** Person, Team, and Seat IDs never change. People are matched across syncs by `Source key`, never by name.
- **Data model** (database Sheet tabs; `*` is a foreign key):
  - Person: ID, Name, Team ID*, Role, Source key
  - Team: ID, Name
  - Seat: ID, Person ID*
- **Not stored in the Sheet:**
  - Team colors: from a 20-color palette, assigned when the app loads
  - Highlighted roles: a list in the code, each with an icon, matched ignoring capitals and extra spaces
  - Desk labels and positions: the desk config file in `app/`, keyed by Seat ID
  - Undo history: app memory only
- Sheet IDs come from Script Properties (`DB_SHEET_ID`, `PEOPLE_SHEET_ID` with the tab name). Never hardcode them.
- Saves are checked on the server too; never rely on hidden UI alone to block viewers.
- Plan files are JSON. Uploading one asks to save the current plan first; people in the file who no longer exist leave their desks empty and are listed in a summary.

## Commands

```bash
node --test              # run rule tests
node scripts/build.js    # build app/ into src/ (the workflows run this)
```

Run the app locally by opening `app/index.html` with VS Code's Live Server.

## Environments

| | Staging | Production |
| --- | --- | --- |
| Owner account | Work Google account (builders' org) | Account in the client's Google Workspace |
| Apps Script project | Puwesto Staging | Puwesto |
| Database Sheet | Test database Sheet | Real database Sheet |
| People Sheet | Sample people Sheet (fake, messy on purpose) | Client's people Sheet |
| Branch | `develop` | `main` |
| Web app access | `ANYONE` | `DOMAIN` |
| Link | `jjepss.github.io/puwesto/staging` | `jjepss.github.io/puwesto` |

Staging renders a `STAGING` badge in the header.

## Git workflow

1. Create `feature/<short-name>` from `main`.
2. Test: open a PR from the feature branch into `develop`. No approval needed; merging deploys to staging. Use this description:
   ```markdown
   ## Testing on staging

   What changed: <one or two lines>

   ## What to check on staging

   - <what to click or look at>
   - <what should happen>
   ```
3. Ship: open a PR from the feature branch into `main`. It needs one approval, and only the repo admin merges. Merging deploys to production.
4. Each feature ships on its own. `develop` is never merged into `main`.
5. More commits on the feature branch after its develop PR need another PR into `develop` to reach staging.

**Branch protection**

- `main`: no deletions, no force pushes, PR with one approval required, updates restricted to the repo admin.
- `develop`: no deletions, no force pushes, PR required with zero approvals. The repo admin can bypass.

**Resetting develop** (repo admin only, when nobody is mid-test):

```bash
git checkout develop
git fetch origin
git reset --hard origin/main
git push --force
```

## CI/CD

On push to `develop`, `deploy-staging.yml`:

1. Installs Node and clasp 3
2. Writes `~/.clasprc.json` from the login secret
3. Writes `.clasp.json` with the environment's script ID
4. Builds `app/` into `src/` and runs the tests; a failing test stops the deploy
5. Runs `clasp push --force`
6. Runs `clasp update-deployment <id>` so the web app URL never changes

`deploy-production.yml` does the same on push to `main` with the `PROD_` secrets, and uses `jq` to switch the manifest's access to `DOMAIN` before pushing.

**Repo secrets**

| Secret | Value |
| --- | --- |
| `STAGING_CLASPRC_JSON` | `~/.clasprc.json` from `clasp login` on the staging owner account |
| `STAGING_SCRIPT_ID` | Staging Apps Script project ID |
| `STAGING_DEPLOYMENT_ID` | Staging web app deployment ID |
| `PROD_CLASPRC_JSON` | `~/.clasprc.json` from `clasp login` on the production owner account |
| `PROD_SCRIPT_ID` | Production Apps Script project ID |
| `PROD_DEPLOYMENT_ID` | Production web app deployment ID |

If a deploy fails on login, the token expired: redo `clasp login` on that environment's account and replace its secret.

## One-time setup (per environment)

1. On the owner account, turn on the Apps Script API at script.google.com/home/usersettings.
2. `npm install -g @google/clasp` and `clasp login` with that account.
3. Create the Apps Script project and the first web app deployment by hand to get its deployment ID.
4. Set `DB_SHEET_ID` and `PEOPLE_SHEET_ID` in the project's Script Properties.
5. Add the environment's three repo secrets.
6. Turn on GitHub Pages from `main`, `/docs`, and point the redirect page at the web app URL.
7. Production only: add warning-only protection to the database Sheet's tabs by hand ("This is managed by Puwesto, edit anyway?").

## Transferring production

Production is tied to one account in the client's Workspace. Before that account is closed, move it to another account there:

1. Repeat the one-time setup on the new account: new Apps Script project, first deployment, and Script Properties.
2. Replace `PROD_CLASPRC_JSON`, `PROD_SCRIPT_ID`, and `PROD_DEPLOYMENT_ID` with the new account's values.
3. Update `docs/index.html` with the new web app link and merge it into `main`.
4. Share the database Sheet and people Sheet with the new account, or transfer their ownership to it.

## Access

```json
"webapp": {
  "executeAs": "USER_ACCESSING",
  "access": "ANYONE"
}
```

- The app runs as the signed-in user, so Google sharing controls access. Sharing or unsharing the database Sheet adds or removes users.
- Viewers of the database Sheet see the seat plan only. Editors can change seats, use the People tab, and run Sync.
- Sync reads the people Sheet as the editor who runs it, so that editor needs at least view access to it.
- Known limits: viewers can open the database Sheet read-only in Drive, and if two editors work at once, the last save wins.
- Users authorize the script on first open and may see an "unverified app" warning.
- GitHub Pages uses redirects, not iframes, because Google sign-in often breaks inside embedded Apps Script apps.

## Do not

- Commit built files in `src/` (`src/Index.html` and the generated `src/*.html`).
- Commit secrets, `.clasprc.json`, or real people data. Sample data must be fake.
- Change the access setting in `src/appsscript.json`; the production workflow handles `DOMAIN`.
- Run `clasp push` or `clasp deploy` by hand. Only the workflows deploy.
- Push to `main` directly or merge `develop` into `main`.

## How to work in this repo

- Follow the current lesson's scope. Build only what was asked; don't add features, refactors, or libraries on the side.
- Prefer the simplest code that works over clever code.
- After writing code, explain it: what each part does and how it ties to the lesson's concepts.
- When adding a rule, add or update its test.
- In docs and comments, avoid em dashes.