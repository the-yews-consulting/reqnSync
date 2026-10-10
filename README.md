# ReqnSync

**The Playwright acceptance suite's dashboard, connected to Azure DevOps.** Dashboard 0.24.0 · runner 1.19.0.

ReqnSync shows how each story's or journey's automated tests stand. From it you can run tests on your machine, record new journeys, write and debug feature files, and write the results back to the work items. Everything runs locally: the dashboard calls ADO directly with your PAT, and a PowerShell runner on `localhost` builds and runs the suite for it.

Feature files are for automated tests only. Tests done by hand live in [ChecknSync](../CheckNSync/), a separate app with its own colours. The **✅ ChecknSync** button in the header opens it once it is set up.

---

## Contents

- [Quick start](#quick-start)
- [Requirements](#requirements)
- [Connecting to ADO](#connecting-to-ado)
- [Environments](#environments)
- [The tabs](#the-tabs)
- [Running tests](#running-tests)
- [Recording and writing journeys](#recording-and-writing-journeys)
- [Features from acceptance criteria](#features-from-acceptance-criteria)
- [The feature editor](#the-feature-editor)
- [Results, evidence and Allure](#results-evidence-and-allure)
- [Quality gate](#quality-gate)
- [Flaky tests](#flaky-tests)
- [Writing results to ADO](#writing-results-to-ado)
- [Test data generator versions](#test-data-generator-versions)
- [Backup and restore](#backup-and-restore)
- [Command line](#command-line)
- [Files in this folder](#files-in-this-folder)
- [Safety and privacy](#safety-and-privacy)
- [Troubleshooting](#troubleshooting)
- [Runner API](#runner-api)

---

## Quick start

1. Start the runner from the repo root:

   ```cmd
   tools\ReqnSync\reqnsync.cmd
   ```

   or, in PowerShell:

   ```powershell
   .\tools\ReqnSync\reqnsync-runner.ps1            # -Port 5078 if 5077 is taken, -NoBrowser to not open a tab
   ```

   It opens the dashboard at `http://localhost:5077/`. **Keep the runner's window open, because it runs the tests.** Ctrl+C stops it, along with any run in progress.
2. Click the **ADO** chip and connect (see [Connecting to ADO](#connecting-to-ado)). The 🖥 Local and 📂 Features tabs work without ADO.
3. Pick the environment in the header (**TST / PREPROD / PROD**).
4. Open **🧰 Setup** to check that this machine has everything a run needs.

**After an update:** the page is read from disk every time you reload it, but the runner is only read when it starts. So stop the runner with Ctrl+C, start it again, then reload the page. The page tells you when the runner is too old for it.

## Requirements

**🧰 Setup** checks each of these and can install most of them:

| What | Needed for |
|---|---|
| PowerShell (Windows PowerShell 5.1 or PowerShell 7) | The runner and every script |
| .NET SDK 8 | Building and running the suite, and the recorder |
| The framework and test data generator packages | The suite builds against them, and ILR scenarios need the test data |
| The suite, built | Runs (a run builds it when needed) |
| Playwright Chromium | Every browser run |
| Azure CLI (`az login`) | Key Vault: the sign-in password and SQL credentials |
| Node.js | Installing Allure 3 (npm) |
| Allure 3 (`npm i -g allure`) | 📊 Allure reports, the report attached to ADO, and quality gates |
| VS Code (optional) | **VS Code** in the editor opens the file at the cursor line |

## Connecting to ADO

Click the **ADO** chip and enter the organisation (`sfa-gov-uk`), the project (`DCT`) and a Personal Access Token.

| PAT scope | For |
|---|---|
| Work Items: Read (Read & write to create or link work items) | Query tabs, ＋ ADO |
| Variable Groups: Read | The test accounts' usernames (groups 866, 944 and 960) |
| Packaging: Read | 🧩 Test data generator: listing and fetching from `dct-pkg` |
| Project and Team: Read | The project list on the sign-in page |

**Remember PAT** keeps the PAT in this browser only; otherwise it lasts until you close the page. The runner never writes it to disk or puts it in a backup. It is passed to a run's process only, as `ADO_PAT`, so the suite can read usernames. **⏏ Sign out** forgets it.

**Usernames on sign-in:** when you connect, and when you switch environment, the dashboard reads that environment's ADO variable group and merges the usernames into its settings file (`appsettings.json` for TST, `appsettings.preprod.json` or `appsettings.prod.json`). Each merged account gets a `UsernameFrom` line saying where it came from, and the previous file is kept as `.bak`. Passwords never come from ADO: they are `kv:` references, read from Key Vault.

## Environments

Pick the environment once, in the header. Every run, recording and result on the page is for that environment, and the page remembers your choice. **⚙** beside a ▶ Run can pick another environment for that one run.

| Environment | Rows that run | Settings | ADO group |
|---|---|---|---|
| TST | tagged `@tst`, or with no environment tag | `appsettings.json` (+ `appsettings.tst.json`) | 866 |
| PREPROD | tagged `@preprod`, or with no environment tag | `appsettings.preprod.json` | 944 |
| PROD | tagged `@prod` only | `appsettings.prod.json` | 960 |

- **Tags** count wherever they are: on the feature, the scenario or an `Examples:` block. So one Outline can hold rows for several environments. The chips on each row (TST PP PROD) show where it runs.
- **Production:** the page turns red, every run asks you to type PROD, and recording is not allowed. Record on TST or PREPROD, then tag the journey `@prod`.
- **`@destructive`** marks a scenario that changes data. Any run that includes one shows a red warning and asks you to confirm, on every environment.

## The tabs

- **🖥 Local:** journeys on this machine, in `Features\Recorded` and `Features\Inbox`, which git ignores. Tick *include suite features with no work item* to list those too.
  - Drag ⠿ to put rows in your own order.
  - Click a column header to sort, and ↺ to go back to your order.
  - The stat boxes filter the table.
- **📂 Features:** every feature in the suite, by area. Run a whole feature or a single scenario.
- **ADO query tabs:** add a saved query (by path, ID or URL) or a one-off WIQL query. Each work item shows:
  - its features, tagged `@story:<id>` / `@bug:<id>` or named in its *Feature File* field;
  - their newest local result;
  - the ADO status.
- **A row's detail:** click a row to see:
  - each scenario with its last result and when it ran;
  - the failing step, with **line N ↗** into the editor;
  - a 💡 diagnosis;
  - the evidence (screenshot, video, trace);
  - **📊 Allure** for that run.

## Running tests

- **▶ Run** on a row runs that feature, or a work item's features. **▶** on a scenario runs only that scenario.
- **▶ Run failing** runs every scenario in the tab whose newest result failed.
- **The Run card** sets the defaults for every run: browser, video, headless, trace, and reusing the sign-in session. **⚙** beside a ▶ Run sets different options for that one run.

| What is run | How |
|---|---|
| One whole feature | `run-journey.ps1 -Feature <file> -Environment <env>`: lint, build when needed, breakpoints, filing |
| Scenarios, several features, or Run failing | One `dotnet test --filter`, built from the generated `.feature.cs`, plus the environment's filter |

**While a run is going:**
- The stage chips (Setup › Lint › Build › Run) turn green, amber or red. Hover one to see the lines behind it.
- The row being run turns green. It turns amber when the run is waiting for you.
- **■ Cancel** stops the whole run.

Only one run can happen at a time. A run started from another tab or browser is followed too.

**Questions from the run** appear in an amber panel:
- at a `#BP` breakpoint: ▶ Resume, ● Record from here, or 🔍 Playwright Inspector;
- while recording: ■ Stop recording;
- when a passing journey can be filed into a suite folder.

## Recording and writing journeys

- **⏺ Record a new journey** (Local tab): give it a name, and optionally a start address and an account to sign in as first. A browser opens; do the journey, then press **■ Stop recording**. The steps become `Features\Recorded\<name>.feature`, which opens in the editor. Recording again replaces the steps but keeps the feature's tags and `#BP` lines.
- **＋ Write a new feature:** starts a feature with the standard opening steps (submission options page, cookies, sign in) and a `#BP`. **▶ Run feature** stops at the `#BP`, and **● Record from here** records the rest in the same browser.
- **＋ ADO:** creates a User Story or Bug for a journey, or links an existing one, then adds `@story:<id>` to the feature. It never changes a work item's State.
- **🗑:** moves a local journey, its generated code and its recording to `_to_delete\local-journeys\`. Nothing is deleted.

## Features from acceptance criteria

**📋 Feature from ACs** drafts a feature from a work item's Acceptance Criteria. Type the work item's ID and press **Read ACs**, or paste the criteria in. **Save and open in the editor** puts the draft in `Features\Recorded`, tagged `@local @story:<id>`.

**Without AI, the step matcher works through the criteria:**
- Given/When/Then criteria become one scenario each.
- Each line is matched to the suite's existing steps:
  - an exact match is used as it is;
  - a close match is used with the criterion's values put in, under a `# AC:` comment to check;
  - with no match, the criterion's own words are kept, and the editor marks the step.
- Prose or bullet criteria become skeleton scenarios.

**With Copilot**, free with your work account:
1. **✨ Copy prompt for Copilot** copies a prompt and opens Microsoft 365 Copilot Chat. The prompt holds the criteria, the house rules, the nearest existing steps and the recorded element names.
2. Paste it into Copilot and send.
3. Copy Copilot's answer, then use **⇩ Paste Copilot's answer**. Steps Copilot could not find are put under a `# NEW STEP` comment.

Either way, the draft is a starting point: run it (▶ or 🐞 Debug) before relying on it.

## The feature editor

**✎ Edit** opens the editor VS Code uses (Monaco), with a list of every feature file in the repo. If Monaco can't load from the internet, you get a plain text box. Every save stamps the file's first line with `# Updated by firstname.lastname on dd/mm/yyyy HH:MM`.

| Toolbar | What it does |
|---|---|
| ▶ Scenario at cursor | Runs the scenario the cursor is in, with `dotnet test`. Breakpoints are ignored. |
| ▶ Run feature (⚙) | Runs the whole file through `run-journey.ps1`, stopping at breakpoints |
| 🐞 Debug (F5) | Stops at breakpoints and failures, with Step, Run to cursor, Retry and a live page panel |
| ✂ Snippets (Ctrl+J) | Shared snippets from `tools\ReqnSync\snippets\`, or a Background or Scenario copied from any feature |
| VS Code | Opens the file in VS Code at the cursor line |
| Save (Ctrl+S) | Keeps line endings and encoding, writes a `.bak`, and refuses if the file changed on disk since you opened it |

- **Step bindings** are checked as Reqnroll checks them. Red means unbound, amber means ambiguous or an unknown element name.
  - Hover a step for its binding.
  - F12 goes to the C#, and Shift+F12 lists every usage.
  - Ctrl+Space lists the steps you can use; Ctrl+. gives the C# for a new step.
- **Examples and tables:**
  - A row with the wrong number of cells, or a `<placeholder>` with no column, is marked.
  - Shift+Alt+F lines up the tables.
  - The `examples-env` snippet adds one Examples block per environment.
- **Breakpoints:** click a step's line number, or press F9, to add or remove a `#BP` line.
- **Run marks and 💡 suggested fixes:** after a run, passed steps get ✓, the failing line gets ✗ with its error, and steps that never ran are greyed. Common failures come with a suggested fix.
- **Debugging** needs runner 1.17.0 or later and `Hooks\EditorDebugHooks.cs`, which does nothing in normal runs or the pipeline.

| Key | What it does |
|---|---|
| Ctrl+S | Save |
| F9 | Add or remove a breakpoint |
| Ctrl+Space | Steps, element names and snippets |
| F12 / Shift+F12 | Go to the step's binding / list every usage |
| Ctrl+. | Quick fixes |
| F8 / Shift+F8 | Next / previous problem |
| Ctrl+J | Snippet list |
| Shift+Alt+F | Line up tables |
| F5 | Debug, or Continue while paused |
| F10 / Ctrl+F10 | Step / Run to cursor |
| Shift+F5 | Stop debugging |
| Ctrl+R | Retry the failed step |
| Shift+F1 | Editor help |

## Results, evidence and Allure

- **Local result:** each scenario's newest result on the chosen environment, read from `runs\<id>\run.json`.
- **Evidence:** a screenshot, video and trace for each scenario. **Open in the Playwright trace viewer** starts the viewer on this machine. A trace holds every value typed, including the sign-in password: never share one.
- **📊 Allure reports:**
  - on a result, that run's own report;
  - **📊 Allure · all runs**, one report over every run;
  - **📊 Feature**, one report for a feature over all its runs on the chosen environment, with each scenario's last 10 results under Retries.
- **Build failures** are explained under the stage chips, for example two features with the same name, a leftover generated file, or compiler errors.

## Quality gate

After each run, the runner checks it against the quality gate (Allure 3), and the run banner shows **✓ gate passed** or **⛔ gate failed**.

| Rule | TST | PREPROD | PROD |
|---|---|---|---|
| `@smoke` / `@critical` scenarios must not fail | 0 failures | 0 failures | 0 failures |
| Share of scenarios that pass | 90% | 98% | 100% |
| Longest single scenario | 5 minutes | 5 minutes | 5 minutes |

The rules are in `tools\ReqnSync\allurerc.mjs`. To check a run from the command line:

```powershell
.\tools\ReqnSync\allure-run.ps1 -RunId <id> -QualityGate -NoReport              # check one run
.\tools\ReqnSync\allure-run.ps1 -RunId <id> -QualityGate -NoReport -FailOnGate  # exit 4 when it fails (pipeline)
```

## Flaky tests

A scenario, or one Examples row, is **🔀 flaky** on an environment when, in its last 10 runs there, it has both passed and failed and its result has changed at least twice. A scenario tagged `@flaky` always counts as flaky.

- **On the dashboard:** 🔀 shows on the scenario and its feature.
- **In Allure:** `allure-run.ps1` marks the flaky results and lists them in `runs\<id>\flaky.json`.
- **In the quality gate:** a flaky scenario's failure is left out of the pass rate.

## Writing results to ADO

`post-run.ps1` writes a run's results to the User Stories its scenarios are tagged with (`@story:<id>`). It is a dry run unless you add `-Apply`.

```powershell
.\tools\ReqnSync\post-run.ps1                                   # latest run, dry run
.\tools\ReqnSync\post-run.ps1 -Apply
.\tools\ReqnSync\post-run.ps1 -RunJson .\runs\<id>\run.json -Apply
```

| Field (Testing tab) | Written |
|---|---|
| Test Status / Test Status Preprod / Test Status Prod | The one for the run's environment: Passing, Failing, Partly failing or Not run |
| Automation Environment | tst, preprod or prod |
| Scenarios, Last Run, Last Run Link, Feature File | A summary of the run |
| Testing Steps | A table of the scenarios, the failure screenshot, and the run's Allure report attached as one `.html` |

**Automation Status** is never changed, because people own it. Traces are never attached. The fields are created once with `setup-ado-process.ps1` (a dry run, then `-Apply`).

## Test data generator versions

**🧩 Test data generator**, on the Repository card, manages the `DC.ILR.<year>.TestDataGenerator.Generators` versions. The defaults are in the suite's csproj (`Tdg2627Version`, `Tdg2526Version`).

- **↻ Versions on dct-pkg** lists the versions on the feed.
- **⬇ Fetch** downloads one into `local-feed`.
- **Use for runs** builds this browser's runs with that version, without changing the csproj. **↺ Back to the default** undoes it.
- **Make default** writes the version into the csproj, keeping a `.bak`.

From the command line: `run-journey.ps1 -Tdg 2627=0.1.31`.

## Backup and restore

**💾 Backup** makes a zip in `backups\`. Choose what goes in:
- dashboard settings (never the PAT);
- local journeys;
- recordings;
- other Local-tab features;
- run history;
- optionally, evidence (large, and traces hold typed passwords).

**Restore** writes the chosen parts back and deletes nothing. It saves a **pre-restore** backup first, so a restore can be undone. **⬇** downloads a backup, and **Restore from a file…** loads one copied from another machine.

> `backups\` is inside the repo (git ignores it), so a backup goes with the repo if the folder is deleted. Download the backups you want to keep with ⬇.

## Command line

| Command | Does |
|---|---|
| `.\run-journey.ps1 -Feature <file> -Environment preprod` | Runs a journey on an environment (lint, build, run, file) |
| `.\run-journey.ps1 -Recorder -Name x -Environment tst` | Records a new journey |
| `.\run-journey.ps1 ... -Tdg 2627=0.1.31` | Builds with another test data generator |
| `.\run-journey.ps1 ... -Browser firefox -Headless` | Browser and headless for one run |
| `.\tools\ReqnSync\post-run.ps1 [-Apply]` | Writes a run's results to its stories |
| `.\tools\ReqnSync\allure-run.ps1 -RunId <id>` | One run's Allure report (`-SingleFile`, `-NoTrace`, `-NoVideo`) |
| `.\tools\ReqnSync\allure-run.ps1 -Feature 'name' -Environment tst` | One feature's report over all its runs, in `allure-features\` |
| `.\tools\ReqnSync\check-ado-usernames.ps1 -Environment prod -HideValues` | Checks an environment's variable group against the accounts |
| `.\tools\ReqnSync\setup-ado-process.ps1 [-Apply]` | Creates the Testing tab's fields in the ADO process |

Scripts that call ADO take the PAT from `$env:REQNSYNC_PAT`, or from `az login`. To set it for one window without it showing on screen:

```powershell
$env:REQNSYNC_PAT = [Net.NetworkCredential]::new('', (Read-Host 'ADO PAT' -AsSecureString)).Password
```

## Files in this folder

| File | What it is |
|---|---|
| `reqnsync.cmd` | Starts the runner (pwsh if installed, otherwise Windows PowerShell) |
| `reqnsync-runner.ps1` | The local web server: serves the dashboard, runs the suite, builds reports, backs up |
| `reqnsync-dashboard.html` | The dashboard |
| `post-run.ps1` | Writes a run's results to its User Stories |
| `allure-run.ps1`, `allurerc.mjs` | Allure reports and the quality gate rules |
| `check-ado-usernames.ps1` | Checks a variable group against the test accounts |
| `setup-ado-process.ps1`, `ado-fields.json` | Creates the Testing tab's fields in the ADO process |
| `snippets\` | Shared editor snippets (commit them so the team gets them) |

## Safety and privacy

- **Runner access:** the runner listens on `localhost` only, with a random token for each session, and only touches files inside the repo.
- **The PAT** is never written to disk by the runner or put in a backup.
- **Writing to ADO** only happens on an explicit action (Create, or `-Apply`), and a work item's State is never changed.
- **Deleting:** nothing is deleted. Removed journeys go to `_to_delete\`, and settings changes keep a `.bak`.
- **Passwords:** traces hold typed passwords, so they are never posted to ADO. Passwords come from Key Vault; usernames come from ADO variable groups.
- **Production** needs a typed confirmation, and runs `@prod` rows only.

## Troubleshooting

| You see | Do |
|---|---|
| "The runner still running is x; this page needs y" | Ctrl+C in the runner's window, start it again, and reload the page. |
| A single scenario runs the whole feature | The log says "scenario not found in the generated .feature.cs": build the suite once, then run it again. |
| "Nothing in … runs on PROD" | Production only runs `@prod` rows. Tag the scenario or its Examples. |
| No tests matched / NO CASES RAN | Every row is tagged for another environment, or the feature has no generated code yet (build it). |
| 401 / TF400813 from a script | `az` is signed in as another account: set `$env:REQNSYNC_PAT` instead. |
| `[config] WARNING … has no value for … ado:` | That account's username is missing from the environment's variable group: add it there. |
| Key Vault errors at the start of a run | Run `az login` with an account that has Key Vault Secrets User on that environment's vault. |
| Restore fails for a test data generator version | 🧩 Test data generator → ⬇ Fetch it, or go back to the default. |
| 📊 Allure says there is no command line | Run `npm i -g allure`, then restart the runner. |
| Something else | **Debug** in the header shows what the page did, and the runner's window shows what it did. |

## Runner API

`reqnsync-runner.ps1` serves the dashboard and a JSON API on `localhost`. Every call needs the session token.

| Area | Calls |
|---|---|
| Repo and features | `GET /api/info`, `GET /api/scan`, `GET/POST /api/feature`, `POST /api/feature/new`, `POST /api/feature/remove`, `GET /api/file`, `POST /api/open-folder` |
| Editor | `GET /api/bindings`, `GET/POST /api/snippets`, `POST /api/snippets/remove` |
| Runs | `POST /api/run`, `POST /api/run/input`, `POST /api/run/cancel`, `GET /api/run/status`, `POST /api/record`, `GET /api/record-options`, `POST /api/trace` |
| Debugging | `GET /api/debug/state`, `POST /api/debug/command`, `GET /api/debug/shot` |
| Reports | `GET /api/allure`, `POST /api/allure/build`, `POST /api/allure/prune`, `GET/POST /api/quality-gate` |
| Setup | `GET /api/prereqs`, `POST /api/prereqs/install`, `GET /api/packages`, `POST /api/packages/feed`, `/fetch`, `/default`, `POST /api/settings/merge-usernames` |
| Backup | `GET /api/backups`, `GET /api/backup`, `POST /api/backup`, `POST /api/backup/restore`, `POST /api/backup/upload` |

---

ReqnSync Dashboard. Author: Russell Ball.
