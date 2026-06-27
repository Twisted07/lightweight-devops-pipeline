# Lightweight DevOps Pipeline Template
#Author: Abdullateef Idris (Twisted)

A production-ready GitHub Actions pipeline template for small projects and startups. Automates build, test, code quality, security scanning, and deployment — with a single configuration file your team fills in once.

> Built as part of research into accessible DevOps practices for resource-constrained teams. See [research background](#research-background) for context.

---

## Why this exists

Enterprise CI/CD pipelines are too complex for most small projects. They require dedicated platform engineers, third-party tool accounts, and ongoing maintenance that small teams cannot afford. This template is deliberately minimal — it gives you the core capabilities that actually matter, without the overhead that doesn't.

**One file to configure. Four stages. Six languages. Three deploy targets.**

---

## What you get

```
Push / PR
   ↓
validate → build → test + quality ──┐
                 → security scan  ──┴─→ CI gate
                                              ↓
                                       staging deploy  (automatic)
                                              ↓
                                       [manual approval]
                                              ↓
                                       production deploy
```

| Stage | What it does |
|---|---|
| **Validate** | Checks `pipeline.config.yml` before wasting any CI minutes |
| **Build** | Installs runtime, installs deps with caching, compiles, uploads artifact |
| **Test + quality** | Unit tests, integration tests, coverage gate, lint, static analysis — in parallel |
| **Security** | Dependency vulnerability audit + secret scanning posture check |
| **Deploy** | Staging auto-deploy → manual approval gate → production deploy |

---

## Supported languages

| Language | Setup | Lint | Static analysis | Dependency audit |
|---|---|---|---|---|
| Node.js | `setup-node` + npm cache | ESLint | ESLint rules | `npm audit` |
| Python | `setup-python` + pip cache | Flake8 | Pylint | `pip-audit` |
| Java | `setup-java` (Temurin) + Maven cache | Checkstyle | SpotBugs | OWASP Dependency-Check |
| Go | `setup-go` + module cache | golangci-lint | `go vet` | `govulncheck` |
| C# | `setup-dotnet` + NuGet cache | dotnet-format | Roslyn analyzers | `dotnet list package --vulnerable` |
| PHP | `setup-php` + Composer cache | PHP_CodeSniffer | PHPStan | `composer audit` |

## Supported deploy targets

| Target | Staging | Production | Required secrets |
|---|---|---|---|
| **Vercel** | `vercel deploy` | `vercel deploy --prod` | `VERCEL_TOKEN`, `VERCEL_ORG_ID`, `VERCEL_PROJECT_ID` |
| **Railway** | `railway up --environment staging` | `railway up` | `RAILWAY_TOKEN` |
| **Heroku** | Auto-creates `appname-staging` | `git push heroku` | `HEROKU_API_KEY` |
| **Render** | Triggers staging service via Render API | Triggers production service via Render API + polls until live | `RENDER_API_KEY` |
| **Custom** | Any shell command | Any shell command | Whatever your command needs |

---

## Getting started

### 1. Use this template

Click **Use this template** on GitHub, or clone and push to your own repo:

```bash
git clone https://github.com/twisted07/lightweight-devops-pipeline my-project
cd my-project
mkdir .github && mv github/* .github && rm -rf github
rm -rf .git && git init && git remote add origin <your-repo-url>
```

### 2. Edit `pipeline.config.yml`

This is the **only file you need to touch**. Open it and fill in your project's values:

```yaml
runtime:
  language: "node"    # node | python | java | go | csharp | php
  version: "24"

build:
  install_command: "npm ci"
  build_command: "npm run build"
  artifact_path: "dist/"

test:
  unit_command: "npm test"
  coverage_command: "npm run coverage"
  coverage_threshold: 70 # set this to your preferred test coverage level –– 100 for maximum coverage and 0 for no coverage

quality:
  enabled: true
  lint_strictness: "warn"       # start with warn, tighten later
  analysis_strictness: "warn"

deploy:
  target: "railway"             # vercel | railway | heroku | custom
  staging:
    auto_deploy: true
  production:
    manual_approval: true
  railway:
    service_id: "your-service-id"
```

Every field has an inline comment explaining what it does and showing examples for all six languages. You should not need to read any other documentation to get started.

### 3. Add secrets to GitHub

Go to **Settings → Secrets and variables → Actions** and add the secrets for your deploy target:

| Target | Secrets to add |
|---|---|
| Vercel | `VERCEL_TOKEN`, `VERCEL_ORG_ID`, `VERCEL_PROJECT_ID` |
| Railway | `RAILWAY_TOKEN` |
| Heroku | `HEROKU_API_KEY` |
| Render | `RENDER_API_KEY` |
| Custom | Whatever your deploy command needs |

### 4. Configure the production approval gate

Go to **Settings → Environments** and create two environments: `staging` and `production`.

On the **production** environment:
1. Check **Required reviewers**
2. Add the GitHub usernames of people who can approve production deploys
3. Optionally add your deploy secrets scoped to this environment

That's it. The pipeline will pause before every production deploy and notify your reviewers. They approve or reject from the GitHub UI or by email — no extra tooling needed.

### 5. Push to `main`

```bash
git add .
git commit -m "chore: add pipeline"
git push origin main
```

The pipeline starts immediately. You can watch it under the **Actions** tab.

---

## Pipeline configuration reference

### `runtime`

```yaml
runtime:
  language: "node"    # node | python | java | go | csharp | php
  version: "24"       # see version examples in pipeline.config.yml
```

### `build`

```yaml
build:
  install_command: "npm ci"       # required
  build_command: "npm run build"  # set to "" if no build step
  artifact_path: "dist/"          # set to "" if no output directory
```

### `test`

```yaml
test:
  unit_command: "npm test"              # required
  integration_command: ""               # leave "" to skip
  coverage_command: "npm run coverage"  # required
  coverage_threshold: 70                # 0–100; set 0 to disable gate
  coverage_report_path: "coverage/"
```

### `quality`

```yaml
quality:
  enabled: true
  lint_strictness: "warn"       # warn → annotates PR; strict → fails pipeline
  analysis_strictness: "warn"   # same
  lint_config_path: ""          # e.g. ".eslintrc.json" — leave "" for defaults
  analysis_level: 5             # 0–8 for PHPStan; 0–10 for Pylint
```

**Recommended approach for new projects:** start with `warn` for both. Fix the findings sprint by sprint, then switch to `strict` once the codebase is clean. This avoids the pipeline blocking all work on day one.

### `security`

```yaml
security:
  dependency_scan: true    # runs native package manager audit
  secret_detection: true   # checks GitHub push protection is enabled
  fail_on_high: true        # fail pipeline on high/critical CVEs
```

All security tooling is GitHub-native. No third-party accounts or tokens required.

### `deploy`

```yaml
deploy:
  target: "railway"              # vercel | railway | heroku | custom
  staging:
    auto_deploy: true            # deploy to staging on every merge to main
    branch: "main"
  production:
    manual_approval: true        # require human approval before production
    approvers: ["alice", "bob"]  # GitHub usernames (for documentation only —
                                 # enforcement is via GitHub Environments)
```

### `notifications`

```yaml
notifications:
  on_failure: true
  on_success: false             # keep this false to avoid noise
  slack:
    enabled: false
    channel: "#deployments"
  email:
    enabled: true               # uses GitHub's built-in email — no config needed
```

### `advanced`

```yaml
advanced:
  timeout_minutes: 30           # per-job timeout; increase for slow test suites
  cancel_in_progress: true      # cancel older runs when a new commit arrives
  runner: "ubuntu-latest"
```

---

## Artifacts and reports

Every pipeline run produces two downloadable artifacts, available under the **Actions** tab → select a run → **Artifacts**:

| Artifact | Contents | Retained for |
|---|---|---|
| `pipeline-report` | HTML report: test results, coverage bar, quality tool summary | 30 days |
| `build-artifact` | Compiled output from your build step | 1 day |

The security and deploy summaries are written directly to the **GitHub Actions job summary page** — visible without downloading anything.

---

## Branch protection setup (recommended)

To prevent merging code that fails CI, add a branch protection rule on `main`:

1. Go to **Settings → Branches → Add rule**
2. Branch name pattern: `main`
3. Check **Require status checks to pass before merging**
4. Search for and add: `CI gate`
5. Check **Require branches to be up to date before merging**

The `CI gate` job is the single status check you need to track — it only passes when both `test` and `security` pass.

---

## Adjusting code quality strictness over time

The quality gate is designed to grow with your project:

**Stage 1 — New project (recommended starting point)**
```yaml
quality:
  lint_strictness: "warn"
  analysis_strictness: "warn"
```
Findings appear as PR annotations but never block work.

**Stage 2 — Stabilising codebase**
```yaml
quality:
  lint_strictness: "strict"
  analysis_strictness: "warn"
```
Lint errors block merging. Analysis findings still warn only.

**Stage 3 — Mature project**
```yaml
quality:
  lint_strictness: "strict"
  analysis_strictness: "strict"
```
Both gates are enforced. Nothing merges with quality issues.

---

## Repository structure

```
.
├── pipeline.config.yml                  ← the only file you edit
└── .github/
    ├── workflows/
    │   ├── ci.yml                       ← CI: validate → build → test → security
    │   └── deploy.yml                   ← Deploy: staging → approval → production
    └── actions/
        ├── setup/
        │   ├── action.yml               ← runtime setup + dependency install + build
        │   └── validate_config.py       ← config validation with clear error messages
        ├── test/
        │   ├── action.yml               ← test + quality orchestration
        │   ├── check_coverage.py        ← coverage parser for all 6 languages
        │   ├── run_quality.py           ← lint + static analysis dispatcher
        │   └── generate_report.py       ← HTML report generator
        ├── security/
        │   ├── action.yml               ← dependency audit + secret scanning
        │   ├── run_dependency_audit.py  ← per-language vulnerability audit
        │   ├── check_secret_scanning.py ← GitHub push protection posture check
        │   └── write_summary.py         ← security job summary writer
        └── deploy/
            ├── action.yml               ← Vercel / Railway / Heroku / custom adapters
            ├── pre_deploy_check.py      ← secret + environment pre-flight check
            └── write_deploy_summary.py  ← deploy job summary writer
```

**Teams interact with one file. Everything else is pre-built infrastructure.**

---

## Design principles

This template makes deliberate trade-offs in favour of small teams:

**Simplicity over completeness.** No canary deploys, no blue-green switching, no distributed tracing. These are correct choices for projects with 1–10 developers and monolithic architectures. You can add them later — starting simple is not the same as staying simple forever.

**Native tooling over third-party integrations.** Every security and quality tool in this pipeline is either built into GitHub Actions or bundled with the language's standard package manager. No Snyk accounts, no SonarCloud tokens, no Datadog agents. This directly reduces the maintenance surface that small teams must manage.

**Stability over speed.** The pipeline prioritises reliable, reproducible runs over raw execution speed. Caching is used aggressively to keep runs fast, but no step is skipped in the name of speed. This reflects findings from the 2024 DORA State of DevOps Report, which identified stability as the primary challenge for small teams adopting CI/CD.

**Fail fast and loud.** Config validation runs before anything else. Security checks run in parallel with tests — not after. Pre-deploy checks run before deploy commands. Errors surface at the earliest possible point with clear, actionable messages — not buried in the middle of a 10-minute log.

**One config file.** A developer joining a project should be able to understand the entire pipeline by reading one file. This template enforces that constraint structurally — all pipeline logic lives in `.github/` and is never edited after initial setup.

---

## Frequently asked questions

**Can I use this with a monorepo?**
The template is designed for single-service projects. Monorepo support would require path filtering in the workflow triggers and separate config files per service — outside the scope of this template's lightweight design goals.

**What if my test command also generates coverage?**
Set `coverage_command` to the same command as `unit_command`. The coverage parser will pick up whatever report format your tool produces.

**Can I disable the coverage gate entirely?**
Yes — set `coverage_threshold: 0` in `pipeline.config.yml`.

**Can I disable code quality checks?**
Yes — set `quality.enabled: false`. The test stage will still run all tests and coverage checks.

**What if I'm on GitHub Free and secret scanning isn't available?**
The `check_secret_scanning.py` script will warn but not block the pipeline. Secret scanning on private repos requires GitHub Team or above.

**How do I add a new deploy target in the future?**
Add a new step to `.github/actions/deploy/action.yml` with `if: inputs.target == 'your-target'`, add the new option to `pipeline.config.yml` under `deploy.target`, and update the validator in `validate_config.py`.

**Can I run the pipeline locally before pushing?**
Yes — install [`act`](https://github.com/nektos/act) and run `act push`. You can also run individual Python helper scripts directly:
```bash
CONFIG_PATH=pipeline.config.yml python3 .github/actions/setup/validate_config.py
```

---

## Metrics this pipeline targets

Based on the DORA (DevOps Research and Assessment) four key metrics:

| Metric | How this pipeline addresses it |
|---|---|
| **Deployment frequency** | Automatic staging deploys on every merge remove the manual step that most delays shipping |
| **Lead time for changes** | Parallel test + security jobs reduce total pipeline duration; caching reduces it further |
| **Change failure rate** | Coverage gates, lint enforcement, dependency audits, and the manual production approval gate all reduce the probability of a bad release reaching production |
| **Time to restore service** | Out of scope for CI/CD — addressed by monitoring and incident response practices |

---

## Research background

This template was built as a research artifact for a study on lightweight DevOps practices for small projects, conducted at Abertay University. The research addresses the finding that small teams disproportionately lack access to automation practices that larger organisations take for granted.

The design is grounded in:
- Kim et al. (2016) — the Three Ways of DevOps (flow, feedback, continual learning)
- Google Cloud DORA (2024) — the 2024 Accelerate State of DevOps Report, particularly its findings on declining delivery stability and the risks of complexity
- Lwakatare et al. (2019) — evidence that small teams benefit from prescriptive, simplified methodologies rather than scaled-down enterprise patterns

The template is released under the MIT licence. Contributions, pilot adoptions, and feedback are welcome.

---

## Adoption guidelines

A detailed adoption guide is available as [`ADOPTION.md`](ADOPTION.md) (also distributed as a Word document alongside the research artefact). The sections below summarise the key points for developers who want a quick reference without opening a separate document.

---

### Should you adopt this template?

| Your situation | Recommended path |
|---|---|
| 1–10 devs, monolith or simple SOA, deploying to Vercel / Railway / Heroku / Render | **Adopt as-is** — you can be running in 30 minutes |
| Supported language but different deploy target | **Adopt + use the custom target** — set `deploy.target: "custom"` and provide your shell command |
| Language not yet supported (Ruby, Rust, Swift) | **Adopt + extend** — add a setup step and quality runner for your language |
| Existing pipeline that is working well | **Skip** — migration cost outweighs benefit unless your current pipeline has reliability problems |
| Microservices (5+ services) or on-premises infra | **Skip** — the template is designed for single-service projects |

---

### Onboarding checklist

Work through these seven steps in order. Each step is self-contained — if something fails, you can diagnose it at that step without undoing subsequent ones.

1. **Clone or copy** — use the GitHub template button, or copy `.github/` and `pipeline.config.yml` into your existing project
2. **Fill in `pipeline.config.yml`** — all seven sections; every field has inline comments with language-specific examples
3. **Add secrets** — go to Settings → Secrets and variables → Actions and add the credentials for your deploy target
4. **Create GitHub Environments** — create `staging` and `production` environments under Settings → Environments
5. **Enable the approval gate** — on the `production` environment, check Required reviewers and add approver usernames
6. **Add branch protection** — Settings → Branches → Add rule → require the `CI gate` status check on `main`
7. **Push and verify** — push your config, watch the Actions tab, and approve the first production deploy

> **Tip:** For a new project, start with `lint_strictness: "warn"` and `coverage_threshold: 50`. You can tighten both values as the codebase matures — see the quality gate progression table below.

---

### Quality gate progression

The quality gate is designed to grow with your project. Switching from `warn` to `strict` too early blocks legitimate work; staying on `warn` indefinitely defeats the purpose.

| Project stage | `lint_strictness` | `analysis_strictness` | `coverage_threshold` |
|---|---|---|---|
| New (0–3 months) | `warn` | `warn` | 50 |
| Stabilising (3–6 months) | `strict` | `warn` | 60 |
| Mature (6+ months) | `strict` | `strict` | 70 |

The recommended transition pattern is to fix all existing lint findings in a single dedicated PR, then switch `lint_strictness` to `strict`. This avoids the disruptive experience of strict mode blocking every open PR at once.

---

### Measuring your baseline (before adoption)

Measure these three DORA metrics from your existing git history before adopting the pipeline so you have a baseline to compare against at 4 weeks and 12 weeks post-adoption.

| Metric | How to measure |
|---|---|
| **Deployment frequency** | Count production deploys in the last 4 weeks using git tags or deploy logs |
| **Lead time for changes** | Average time from first branch commit to production deploy for 5 recent features |
| **Change failure rate** | % of deploys in the last 4 weeks that required a hotfix or rollback within 24 hours |

```bash
# Count deploys from git tags in the last 4 weeks
git tag --sort=-creatordate | head -20

# First commit on a branch (to calculate lead time start)
git log --oneline origin/main..feature/your-branch | tail -1
```

---

### Common failure patterns

| Symptom | Fix |
|---|---|
| Validator fails: `language not supported` | Check spelling — valid values: `node \| python \| java \| go \| csharp \| php` |
| Coverage check fails: `could not parse report` | Run coverage command locally and verify it creates a file at `coverage_report_path` |
| Deploy fails: `secret not set` | Add the secret to the GitHub **Environment** (not just repo level) under Settings → Environments |
| Production deploy never triggers | Settings → Environments → production → enable Required reviewers and add at least one reviewer |
| No quality annotations on PR | Working as intended — annotations only appear when issues are found |
| `act` fails locally with `image not found` | Run: `act -P ubuntu-latest=catthehacker/ubuntu:act-latest` |

---

### Running the validator locally

Before pushing, validate your config without consuming CI minutes:

```bash
pip install pyyaml
CONFIG_PATH=pipeline.config.yml python3 .github/actions/setup/validate_config.py
```

Errors appear as annotated messages with the exact field name and a description of what is wrong. Fix all errors before pushing.

---

### Extending the template

**Adding a deploy target:** Add a conditional step to `.github/actions/deploy/action.yml` with `if: inputs.target == 'your-target'`, add the target to `SUPPORTED_TARGETS` in `validate_config.py`, and add a sub-section in `pipeline.config.yml`.

**Adding a language:** Add a setup step in `setup/action.yml`, a quality runner function in `test/run_quality.py`, a coverage parser in `test/check_coverage.py`, and the language identifier to `SUPPORTED_LANGUAGES` in `validate_config.py`. Each extension is isolated — adding a new language has no effect on existing language paths.

**Adding performance testing:** Add a job to `ci.yml` after the `test` job. Use a lightweight tool (`k6`, Playwright, `go test -bench`) and store the threshold in `pipeline.config.yml` under a new `performance` section. Do not add performance testing to the main CI path until you have stable baseline numbers — a flaky gate is worse than no gate.

---

## Licence

MIT — see [LICENSE](LICENSE) for details.
