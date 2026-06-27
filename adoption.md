**Lightweight DevOps Pipeline Template**

Evidence-Based Adoption Guidelines

_For developers adopting the template in small projects and startups_

# **Introduction**

These guidelines accompany the Lightweight DevOps Pipeline Template - a GitHub Actions-based CI/CD pipeline designed for teams of one to ten developers. They cover three things: how to decide whether and how to adopt the template; how to onboard a project step by step; and how to diagnose and fix the most common failure patterns.

The guidelines are written for developers, not DevOps engineers. No prior CI/CD experience is required. Every instruction is specific - not 'configure your environment' but 'go to Settings → Environments → production → check Required reviewers.'

_💡 If you have already cloned the template and just want to get running, skip to Section 2: Step-by-Step Onboarding._

# **1\. Decision framework - should you adopt this template?**

Not every project needs this pipeline, and not every project should adopt it unchanged. The table below maps your project's characteristics to a recommended adoption path.

| **Criterion**          | **Adopt as-is**                 | **Adopt + extend**               | **Skip for now**                        |
| ---------------------- | ------------------------------- | -------------------------------- | --------------------------------------- |
| **Team size**          | 1-10 developers                 | 10-20 developers                 | 20+ developers                          |
| **Architecture**       | Monolithic or simple SOA        | Modular monolith or 2-3 services | Microservices (5+ services)             |
| **Language**           | Node, Python, Java, Go, C#, PHP | Any of the above + other lang    | Language not yet supported              |
| **Deploy target**      | Vercel, Railway, or Heroku      | Custom target with shell command | On-premises or complex cloud            |
| **CI/CD experience**   | None to moderate                | Moderate                         | Experienced team with existing pipeline |
| **Current automation** | None or ad-hoc scripts          | Partial (some stages only)       | Full pipeline already in place          |
| **Timeline pressure**  | High - need something now       | Medium                           | Low - time to build bespoke             |

_Table 1: Adoption decision framework_

## **1.1 When to adopt as-is**

The template is designed for exactly this scenario: a small team with no existing automation, working on a single codebase, deploying to a PaaS provider. If this describes your project, you can be fully operational within 30 minutes. The only decisions you need to make are which language, which deploy target, and how strict you want the quality gate to be at the start.

_💡 New projects benefit most from adopting the pipeline on day one, before technical debt accumulates and before manual deployment habits become entrenched._

## **1.2 When to extend the template**

If your project uses a language not yet supported (Ruby, Rust, Swift), or deploys to a target not covered by the built-in adapters (AWS, GCP, Azure), use the custom deploy target and extend the quality stage. Adding a new language requires:

- Adding a setup step in .github/actions/setup/action.yml with the appropriate actions/setup-X action
- Adding a quality runner function in .github/actions/test/run_quality.py
- Adding a coverage parser function in .github/actions/test/check_coverage.py
- Updating SUPPORTED_LANGUAGES in .github/actions/setup/validate_config.py

Each extension is isolated - adding Go support, for example, has no effect on the Node.js or Python paths.

## **1.3 When to skip the template**

Teams with an existing CI/CD pipeline that is working well should not adopt this template. Migration cost - retraining, reconfiguring branch protection, updating developer documentation - outweighs the benefit unless the existing pipeline has serious reliability or maintenance problems.

Teams deploying to on-premises infrastructure or complex cloud environments (Kubernetes clusters, multi-region AWS setups) will need a bespoke pipeline. The custom deploy target can be a stepping stone, but the template's built-in adapters will not cover these cases adequately.

_⚠️ Do not adopt the template if your project already has a functioning, well-maintained pipeline. Replacing working automation is not a productivity improvement._

# **2\. Step-by-step onboarding**

This section walks through complete adoption from zero to a working pipeline. Each step is self-contained - if something fails, you can diagnose it at that step without having to undo subsequent steps.

Estimated time: 20-30 minutes for a straightforward project on a supported language and deploy target.

## **Step 1 - Clone or use the template**

If the template is published as a GitHub Template Repository, click Use this template on the GitHub repository page and create a new repository from it. This copies all files without the commit history.

Alternatively, clone and re-initialise manually:

git clone <https://github.com/Twisted07/lightweight-devops-pipeline> my-project

cd my-project

mkdir .github && mv github/\* .github && rm -rf github

rm -rf .git && git init

git remote add origin &lt;your-new-repo-url&gt;

If you are adding the pipeline to an existing project, copy only the .github/ directory and the pipeline.config.yml file into your project root.

_💡 Copying into an existing project is the most common adoption path. The pipeline files do not interfere with your application code._

## **Step 2 - Fill in pipeline.config.yml**

Open pipeline.config.yml in your editor. This is the only file you need to edit. Work through each section top to bottom:

### **Project section**

Set project.name to your project's name. This appears in notifications and the deploy summary. The default value 'my-project' will trigger a validator warning on every run until you change it.

project:

name: "my-app"

### **Runtime section**

Set runtime.language to your primary language and runtime.version to the specific version you are targeting. Use the version string your platform expects - not a major.minor range, but a specific version like '20' for Node.js 20 LTS or '3.12' for Python 3.12.

runtime:

language: "node" # node | python | java | go | csharp | php

version: "20"

_⚠️ Do not use version: 'latest'. Pinning to a specific version ensures that your pipeline behaves identically on every run and does not break when a new runtime version is released._

### **Build section**

Set the install_command to the command that installs your project's dependencies. Set build_command to the command that compiles or bundles your code - or leave it as an empty string if your language has no compile step (Python, for example). Set artifact_path to the directory your build command produces, or leave it empty.

Language-specific examples for **install_command**:

- Node.js: npm ci
- Python: pip install -r requirements.txt
- Java: mvn -B dependency:resolve
- Go: go mod download
- C#: dotnet restore
- PHP: composer install --no-interaction --prefer-dist

### **Test section**

Set unit_command to the command that runs your unit tests. If you have integration tests, set integration_command - otherwise leave it empty. Set coverage_command to the command that generates your coverage report. The pipeline will parse whichever report format your language produces automatically.

Set coverage_threshold to the minimum percentage you want to enforce. For a new project starting with limited tests, 50 is a reasonable starting point. The default is 70, which aligns with DORA research recommendations for mature projects. Set it to 0 to disable the coverage gate entirely.

_💡 Start with a lower threshold than you think you need. You can raise it as the test suite grows. A threshold that blocks all work on day one is counterproductive._

### **Quality section**

For new projects, set both lint_strictness and analysis_strictness to 'warn'. This means quality findings are posted as annotations on pull requests but never block merging. You can see what the tools are catching without your team being stopped by it. Switch to 'strict' once the codebase is clean.

quality:

enabled: true

lint_strictness: "warn"

analysis_strictness: "warn"

### **Deploy section**

Set deploy.target to your deployment platform. Then fill in the corresponding sub-section for that platform - service ID for Railway, app name for Heroku, project and org IDs for Vercel. Leave the other platform sub-sections as they are.

Set deploy.production.manual_approval to true. This is the default and is strongly recommended - it means every production deploy requires a human to click Approve in the GitHub Actions UI before the deployment runs.

Leave **deploy.production.approvers** as an empty list unless you want to document approver names for reference. Enforcement is handled by GitHub Environments, not by this field.

## **Step 3 - Add secrets to GitHub**

Go to your repository on GitHub. Navigate to Settings → Secrets and variables → Actions → New repository secret. Add the secrets for your chosen deploy target:

| **Deploy target** | **Required secrets**                           | **Where to get them**                                   |
| ----------------- | ---------------------------------------------- | ------------------------------------------------------- |
| **Vercel**        | VERCEL_TOKEN, VERCEL_ORG_ID, VERCEL_PROJECT_ID | Vercel dashboard → Settings → Tokens / Project Settings |
| **Railway**       | RAILWAY_TOKEN                                  | Railway dashboard → Account Settings → Tokens           |
| **Heroku**        | HEROKU_API_KEY                                 | Heroku dashboard → Account Settings → API Key           |
| **Custom**        | Whatever your deploy command needs             | Wherever your target platform issues credentials        |

_Table 2: Required secrets by deploy target_

_⚠️ Secrets added at the repository level are available to all workflows. For stronger isolation, add secrets at the Environment level instead - see Step 4._

## **Step 4 - Configure GitHub Environments**

GitHub Environments are what makes the manual approval gate work. Without this step, the pipeline will deploy to production automatically without waiting for approval.

Go to Settings → Environments in your repository. You need to create two environments:

### **Creating the staging environment**

- Click New environment
- Name it exactly: staging (case-sensitive)
- Click Configure environment
- Add your deploy secrets here if you want environment-scoped secrets
- Save the environment

### **Creating the production environment**

- Click New environment
- Name it exactly: production (case-sensitive)
- Click Configure environment
- Check Required reviewers
- Add the GitHub usernames of people who can approve production deploys
- Optionally set a wait timer (e.g. 5 minutes) as a cooling-off period
- Add your production deploy secrets here
- Save the environment

_💡 The environment names must match exactly. The deploy.yml workflow references 'staging' and 'production' by name. A typo here means the approval gate silently does not apply._

## **Step 5 - Set up branch protection**

Branch protection ensures that no code merges to main without passing the full CI pipeline. Without this, the pipeline runs but does not enforce anything.

- Go to Settings → Branches → Add branch protection rule
- Set Branch name pattern to: main
- Check Require status checks to pass before merging
- In the search box, type CI gate and select it from the list
- Check Require branches to be up to date before merging
- Click Create

_💡 The CI gate job is the single status check to configure. It only passes when both the test and security jobs pass, so you do not need to add both individually._

## **Step 6 - Push and verify**

Commit your pipeline.config.yml changes and push to main:

git add pipeline.config.yml

git commit -m "chore: configure pipeline"

git push origin main

Go to the Actions tab in your repository. You should see two workflow runs start: CI and Deploy. Watch the CI run first. The validate job runs in under 30 seconds and will surface any configuration errors immediately with clear, annotated error messages.

If the CI run passes, the Deploy run will trigger automatically. The staging deploy runs without intervention. The production deploy will pause and send approval notifications to your configured reviewers.

## **Step 7 - Approve the first production deploy**

In the Actions tab, click on the Deploy workflow run. You will see the deploy-production job showing a yellow pending indicator with the message 'Waiting for review.' Click Review deployments, select production, add an optional comment, and click Approve and deploy.

The production deploy runs and posts a summary to the job summary page showing the deploy URL, commit, timestamp, and triggered-by username.

_💡 On the first run you may want to verify the staging deploy worked correctly before approving production. Check the staging URL (posted as a PR comment or visible in the deploy summary) before clicking Approve._

# **3\. Quality gate progression**

The code quality gate is designed to grow with your project. Starting too strict blocks work; staying too loose defeats the purpose. The table below gives a recommended progression based on project maturity.

| **Project stage**            | **Recommended settings**                                                   | **Rationale**                                                                              |
| ---------------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **New project (0-3 months)** | lint_strictness: warn analysis_strictness: warn coverage_threshold: 50     | Build awareness without blocking work. Teams can see issues without being stopped by them. |
| **Stabilising (3-6 months)** | lint_strictness: strict analysis_strictness: warn coverage_threshold: 60   | Enforce style consistency. Tighten coverage as test suite matures.                         |
| **Mature (6+ months)**       | lint_strictness: strict analysis_strictness: strict coverage_threshold: 70 | Full enforcement. Both gates block merging. 70% coverage is the DORA-aligned minimum.      |

_Table 3: Recommended quality gate progression by project stage_

## **3.1 Interpreting quality annotations**

When lint_strictness or analysis_strictness is set to 'warn', findings appear as annotations directly on the diff in your pull request - the same way code review comments appear, but automated. Each annotation shows the file, line number, rule name, and a brief description of the issue.

When set to 'strict', the same annotations appear but the pipeline also fails, blocking the merge. The team sees exactly the same information in both modes - the only difference is whether the pipeline stops.

A common pattern is to fix all existing lint findings in a single dedicated PR, then switch to 'strict' mode. This avoids the disruptive experience of switching to strict mode and immediately blocking every open PR.

## **3.2 Providing a lint configuration file**

By default, each tool runs with its built-in defaults. For most projects this is a reasonable starting point. To customise rules, create a configuration file for your language's tool and set quality.lint_config_path in pipeline.config.yml:

- Node.js: .eslintrc.json
- Python: setup.cfg (Flake8 config under \[flake8\])
- Java: checkstyle.xml
- Go: .golangci.yml
- C#: .editorconfig
- PHP: phpcs.xml

_💡 Start without a custom lint config. Let the tool run with defaults for two weeks and observe what it flags. Then create a config that accepts the patterns your team intentionally uses and rejects the patterns that genuinely indicate problems._

# **4\. Measuring your baseline before adoption**

To evaluate the pipeline's impact, you need to measure your delivery performance before adoption. This takes about an hour and uses only your existing git history and deployment records - no external tools required.

| **DORA metric**           | **How to measure baseline**                                                                       | **Expected post-pipeline improvement**                                                                   |
| ------------------------- | ------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| **Deployment frequency**  | Count production deploys in the 4 weeks before adoption (from git tags or deploy logs)            | ≥50% increase. Auto-staging removes the manual bottleneck that delays most small-team deploys.           |
| **Lead time for changes** | Average time from first commit on a branch to production deploy (use git log + deploy timestamps) | ≥30% reduction. Parallel CI jobs and caching cut pipeline time; auto-staging removes waiting.            |
| **Change failure rate**   | % of deploys that required a hotfix or rollback in the 4 weeks before adoption                    | Maintained or improved. Coverage gate, dependency audit, and manual approval reduce failure probability. |

_Table 4: DORA baseline measurement guide_

## **4.1 Measuring deployment frequency**

Count the number of times code was deployed to production in the four weeks before pipeline adoption. Use git tags, deployment log entries, or release notes as your source. Divide by four to get deploys per week.

After adoption, measure the same metric at four weeks and twelve weeks post-adoption. The pipeline auto-deploys to staging on every merge to main and removes the manual production deploy step - most teams see deployment frequency increase significantly within the first two weeks.

\# Count deploys from git tags in the last 4 weeks

git tag --sort=-creatordate | head -20

## **4.2 Measuring lead time**

Lead time is the time from the first commit on a feature branch to the moment that code reaches production. For a rough baseline, pick five recent features and calculate the average time from the branch creation date to the production deploy date using git log and your deploy records.

\# First commit on a branch

git log --oneline origin/main..feature/my-branch | tail -1

## **4.3 Measuring change failure rate**

Review your git history and issue tracker for the four weeks before adoption. Count how many production deploys required a hotfix commit, a rollback, or an incident response within 24 hours. Divide by total deploys to get the failure rate as a percentage.

This is the metric to watch most carefully after adoption. The pipeline adds quality gates specifically to reduce this number. If it increases after adoption, it is a signal that the gates are not catching the types of failure your project actually experiences - review the coverage threshold and consider adding integration tests.

# **5\. Troubleshooting**

This section covers the most common failure patterns and their fixes. Each entry is structured as symptom → cause → fix so you can scan the symptom column and jump directly to the relevant row.

| **Symptom**                                          | **Likely cause**                                          | **Fix**                                                                                                    |
| ---------------------------------------------------- | --------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| **Validation fails with 'language not supported'**   | Typo in runtime.language                                  | Check spelling: node \| python \| java \| go \| csharp \| php                                              |
| **Build fails: 'install command not found'**         | Wrong package manager for the language                    | Check build section examples in pipeline.config.yml comments                                               |
| **Coverage check fails: 'could not parse report'**   | Coverage command not generating a report file             | Run coverage command locally and verify it creates a report at coverage_report_path                        |
| **Dependency audit fails with high CVEs**            | Outdated package with a known vulnerability               | Run audit locally (npm audit / pip-audit etc.), update the package, or add a suppression for accepted risk |
| **Deploy step fails: 'secret not set'**              | Secret added to repo but not to the GitHub Environment    | Go to Settings → Environments → your environment name → add the secret there, not just at repo level       |
| **Production deploy never triggers**                 | GitHub Environment not configured with required reviewers | Settings → Environments → production → enable Required reviewers and add at least one reviewer             |
| **Pipeline runs but PR has no quality annotations**  | lint_strictness is warn but no findings exist             | Working as intended - annotations only appear when issues are found                                        |
| **act (local testing) fails with 'image not found'** | act needs a Docker image for the runner                   | Run: act -P ubuntu-latest=catthehacker/ubuntu:act-latest                                                   |

_Table 5: Common failure patterns and fixes_

## **5.1 Running the validator locally**

The config validator can be run locally before pushing to catch errors without consuming CI minutes:

pip install pyyaml

CONFIG_PATH=pipeline.config.yml python3 .github/actions/setup/validate_config.py

The validator outputs GitHub-style error annotations that are readable in a terminal. Fix all errors before pushing.

## **5.2 Testing the pipeline locally with act**

act is an open-source tool that runs GitHub Actions workflows locally using Docker. Install it and run:

act push -P ubuntu-latest=catthehacker/ubuntu:act-latest

This runs the full CI workflow locally. It is slower than running in GitHub Actions but useful for debugging workflow YAML syntax errors before pushing.

_⚠️ act does not have access to your GitHub secrets. Deploy jobs will fail locally. Use act to test the build, test, and security stages only._

## **5.3 Reading pipeline annotations in pull requests**

When the pipeline finds lint or security issues, it posts inline annotations on the changed files in your pull request. To see them, go to the Files changed tab in your PR. GitHub shows annotations as yellow (warning) or red (error) markers on specific lines. Click the marker to see the rule name and description.

If the pipeline fails but you see no annotations, the failure is in the test or build stage rather than quality or security. Go to the Actions tab, click the failed run, and expand the failing job's steps to read the log output.

# **6\. Extending the template**

The template is designed to be extended without modifying the core workflow files. All extension points are clearly defined.

## **6.1 Adding a new deploy target**

To add a deploy target beyond Vercel, Railway, Heroku, and custom:

- Open .github/actions/deploy/action.yml
- Add a new input for any credentials needed
- Add a deploy step with if: inputs.target == 'your-target'
- Set echo "url=..." >> \$GITHUB_OUTPUT in your deploy step
- Add the new target to SUPPORTED_TARGETS in validate_config.py
- Add a new sub-section in pipeline.config.yml with inline comments

The capture-url step at the end of the action will automatically pick up the URL output from your new step.

## **6.2 Adding a new notification channel**

The pipeline currently supports Slack (via webhook) and GitHub's built-in email. To add a different channel - Microsoft Teams, Discord, PagerDuty - add a step to both ci.yml and deploy.yml that runs on: always() and calls your notification API using a curl command or a marketplace action. Store the webhook URL as a GitHub secret.

## **6.3 Adding performance testing**

For projects that need basic performance testing, add a new job to ci.yml that runs after the test job. Use a lightweight tool appropriate for your stack - k6 for HTTP APIs, Playwright for browser performance, or go test -bench for Go. Gate the job on a configurable threshold stored in pipeline.config.yml under a new performance section.

_⚠️ Do not add performance testing to the main CI path until you have stable baseline numbers. A flaky performance gate is worse than no gate - it blocks legitimate work and trains the team to ignore pipeline failures._

## **6.4 Adding container builds**

If your project needs to build and push a Docker image, add a job to ci.yml after the build job. Use docker/build-push-action with GitHub Container Registry (ghcr.io) as the registry - it requires only the GITHUB_TOKEN secret that is already available in every workflow, with no additional credentials to manage.

# **7\. Ongoing maintenance**

One of the core design goals of this template is minimal ongoing maintenance. The following are the only maintenance activities a team should expect to perform.

## **7.1 Keeping action versions current**

GitHub Actions marketplace actions (actions/setup-node@v4, actions/checkout@v4 etc.) release new major versions periodically. Subscribe to the Dependabot alerts for your repository - GitHub will automatically open pull requests to update action versions when new major releases are available. Review and merge these PRs promptly; major version bumps occasionally include breaking changes that affect workflow behaviour.

_💡 Dependabot for GitHub Actions is enabled by default in public repositories. For private repositories, go to Settings → Code security → Dependabot version updates and enable it._

## **7.2 Rotating secrets**

Pipeline secrets - API keys and deploy tokens - should be rotated at least annually or immediately if there is any suspicion of compromise. Generate a new token in the relevant platform's dashboard, update the GitHub secret, and verify the next pipeline run succeeds before revoking the old token.

## **7.3 Reviewing the coverage threshold**

Review coverage_threshold in pipeline.config.yml every quarter. As the test suite grows, the threshold should increase. A threshold that has not changed in a year is probably not being maintained. The target for a mature project is 70-80%; above 80% the marginal value of additional coverage typically diminishes.

## **7.4 Monitoring CI minutes usage**

GitHub's free tier provides 2,000 CI minutes per month for private repositories. Check your usage monthly under Settings → Billing. If you are approaching the limit, the most effective optimisations are: enabling dependency caching (already configured in the template), reducing the coverage_command execution time by running coverage on a subset of tests, or reducing the timeout_minutes value in the advanced section.

# **Quick reference**

## **Onboarding checklist**

- Clone template or copy .github/ into existing project
- Fill in pipeline.config.yml - all seven sections
- Add deploy secrets to GitHub (Settings → Secrets → Actions)
- Create staging and production GitHub Environments
- Enable Required reviewers on production environment
- Add CI gate branch protection rule on main
- Push, watch CI run, approve first production deploy
- Measure DORA baselines before and after adoption

## **Quality gate progression**

- Week 0: lint_strictness: warn | analysis_strictness: warn | coverage_threshold: 50
- Month 3: lint_strictness: strict | analysis_strictness: warn | coverage_threshold: 60
- Month 6+: lint_strictness: strict | analysis_strictness: strict | coverage_threshold: 70

## **Key file locations**

- pipeline.config.yml - the only file you edit
- .github/workflows/ci.yml - CI orchestration (do not edit)
- .github/workflows/deploy.yml - deploy orchestration (do not edit)
- .github/actions/setup/ - runtime setup and config validation
- .github/actions/test/ - test, coverage, quality, and HTML report
- .github/actions/security/ - dependency audit and secret scanning
- .github/actions/deploy/ - Vercel / Railway / Heroku / custom adapters

## **Support and contributing**

This template is released under the MIT licence. Bug reports, language additions, and deploy target adapters are welcome via GitHub Issues and Pull Requests. When reporting a pipeline failure, include the run URL, the pipeline.config.yml content (with secrets redacted), and the full error output from the failing step.
