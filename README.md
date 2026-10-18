# Puwesto

An office seating planner. Click a desk, pick who sits there, and desks take on team colors so team clusters show on the floor map.

Built as a Google Apps Script web app with Google Sheets as its data store, in plain HTML, CSS, and JavaScript.

## How it works

- The page runs in the browser and talks to Apps Script through `google.script.run`.
- Apps Script reads and writes a **database Sheet** (Person, Team, Seat) and reads the client's **people Sheet** during Sync.
- Access follows Google sharing: viewers of the database Sheet can see the plan, editors can change it.
- Plans export to PNG, PDF, or a spreadsheet laid out like the floor map.

## Tech stack

| Layer | Tool |
| --- | --- |
| Frontend | Plain HTML, CSS, JavaScript |
| Backend | Google Apps Script |
| Storage | Google Sheets |
| Export | html-to-image, jsPDF, ExcelJS |
| Deployment | GitHub Actions + clasp |
| Tests | Node test runner |

## Run locally

Requires VS Code, Node, and Git. No Google login needed; locally the app uses sample data.

```bash
git clone https://github.com/jjepss/puwesto.git
cd puwesto
node --test
```

Open `app/index.html` with the VS Code Live Server extension.

## Environments

| | Branch | Link |
| --- | --- | --- |
| Staging | `develop` | [jjepss.github.io/puwesto/staging](https://jjepss.github.io/puwesto/staging) |
| Production | `main` | [jjepss.github.io/puwesto](https://jjepss.github.io/puwesto) |

Merging into a branch deploys it automatically.

## Contributing

1. Branch from `main`.
2. Open a PR into `develop` to test on staging.
3. Open a PR into `main` to ship; it needs one review.

Conventions, deployment details, setup, and how to transfer production to another account are in [AGENTS.md](AGENTS.md).