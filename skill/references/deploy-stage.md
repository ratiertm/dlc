# DEPLOY Stage Adapter

This reference defines the complete DEPLOY Stage workflow -- Stage 5 of the dev-lifecycle pipeline. When Claude reads this file, it follows the instructions below to guide the user through deploying their feature using project-type-specific checklists.

## When This Runs

- User says "deploy", "ship", "release", or similar
- `state.json` `current.stage = 5` (or transitioning from 4 to 5)
- SKILL.md Stage 5 section directs here via `Read: $CLAUDE_SKILL_DIR/references/deploy-stage.md`

## Prerequisites

- `.lifecycle/` directory initialized (from PLAN stage)
- `state.json` and `manifest.json` exist
- COMMIT stage must have outputs -- gate 4_to_5 must pass (`artifacts.4_commit.outputs.length > 0`)

## Step 0: Stage Initialization

Before any deploy work, verify the pipeline state and check mode-based skipping.

**Actions:**

1. Read `.lifecycle/state.json` and verify:
   - `current.feature` has a value
   - `current.stage` = `5` (or transition from 4 to 5)

2. Check gate condition (4_to_5):
   - Read `.lifecycle/manifest.json`
   - Verify: `artifacts.4_commit.outputs.length > 0`
   - If gate fails: set `current.status = "blocked"`, display missing artifacts, STOP

3. Check mode-based skip:
   - Read `current.mode` from `state.json`
   - If mode is `feature`: offer skip -- "Feature mode allows skipping DEPLOY. Skip? (yes/no)"
   - If user chooses skip: register `{ "type": "skipped", "path": "N/A", "status": "skipped" }` in `artifacts.5_deploy.outputs`, set `artifacts.5_deploy.completed_at`, set `current.status = "skipped"`, announce skip, STOP
   - If mode is `hotfix`, `release`, or `milestone`: proceed normally

4. Update `state.json`:
   - `current.stage` = `5`
   - `current.stage_name` = `"DEPLOY"`
   - `current.status` = `"in_progress"`
   - `progress.current_stage_started_at` = current ISO 8601 timestamp

## Step 1: Project Type Resolution

**Purpose:** Determine the project type to select the appropriate deploy checklist.

**Actions:**

1. Read `state.json` `project.detected_type`
2. If `project.detected_type` exists and `project.confirmed` is `true`:
   - Use the confirmed type
   - Display: "Using confirmed project type: {type}"
3. If `project.detected_type` is missing or `project.confirmed` is `false`:
   - Run detection per `$CLAUDE_SKILL_DIR/references/project-detection.md`
   - Present detected type to user for confirmation
   - Store confirmed type in `state.json`
4. For multi-type projects, identify the deployable component(s) and ask user which to deploy

## Step 2: Deploy Checklist

**Purpose:** Present a project-type-specific deploy checklist for the user to follow.

Select the appropriate checklist based on confirmed project type. Each checklist has 3-5 high-level steps. Claude presents the checklist and tracks completion -- Claude does NOT run production deploy commands directly.

### node-web/next (Next.js)

1. Build production bundle: `npm run build` (or `next build`)
2. Configure environment variables on hosting platform (Vercel, AWS, etc.)
3. Deploy to platform (Vercel auto-deploy, `vercel --prod`, or custom CI/CD)
4. Verify build output and deployment URL

### node-web/react (React SPA)

1. Build production bundle: `npm run build`
2. Configure environment variables for production API endpoints
3. Upload build output to static hosting (S3, Netlify, Vercel, Firebase Hosting)
4. Configure CDN and custom domain if applicable

### node-web/express (Express API)

1. Ensure production dependencies are locked (`package-lock.json`)
2. Configure environment variables on server/platform
3. Deploy to platform (Docker, PM2, AWS ECS, Railway, Render)
4. Verify health endpoint responds
5. Configure reverse proxy / load balancer if applicable

### flutter

1. Build release: `flutter build apk` (Android) / `flutter build ios` (iOS) / `flutter build web`
2. Sign release build with production certificates
3. Upload to distribution platform (Play Store, App Store, Firebase App Distribution)
4. Submit for review if app store release

### python/fastapi

1. Freeze dependencies: `pip freeze > requirements.txt` or lock `pyproject.toml`
2. Configure environment variables on server/platform
3. Deploy with production server (uvicorn behind gunicorn, Docker, or platform deploy)
4. Verify API docs endpoint (`/docs`) is accessible or disabled per policy

### python/django

1. Run `python manage.py collectstatic`
2. Configure `ALLOWED_HOSTS`, `SECRET_KEY`, database URL for production
3. Run migrations: `python manage.py migrate`
4. Deploy with production server (gunicorn + nginx, Docker, or platform deploy)
5. Verify admin panel access if applicable

### rust

1. Build release binary: `cargo build --release`
2. Configure environment variables or config files
3. Deploy binary to server (scp, Docker, or CI/CD artifact)
4. Set up systemd service or container orchestration

### go

1. Build binary: `go build -o app .`
2. Configure environment variables or config files
3. Deploy binary to server (Docker, Kubernetes, or direct)
4. Configure health check endpoint

### ios

1. Archive build via Xcode: Product > Archive
2. Validate and upload to App Store Connect
3. Configure TestFlight or App Store submission
4. Submit for App Review

### android-jvm

1. Build release APK/AAB: `./gradlew assembleRelease` or `bundleRelease`
2. Sign with production keystore
3. Upload to Google Play Console
4. Configure release track (internal, beta, production)

### generic

1. Ask user: "Describe your deploy process (target platform, build steps, deploy method)"
2. Create a custom checklist from user's description
3. Track each step to completion
4. Record deploy details (URL, version, platform)

## Step 3: Deploy Execution

**Purpose:** Guide the user through the selected checklist, tracking progress.

**Actions:**

1. Present the selected checklist to the user
2. For each step:
   - Display the step instruction
   - Wait for user to confirm completion or report issues
   - Record step status (done / skipped / failed)
3. If a step fails:
   - Ask user if they want to retry, skip, or abort
   - If abort: set `current.status = "blocked"`, STOP
4. After all steps complete, collect deploy details:
   - Deploy URL (if web/API)
   - Version or build number
   - Platform/environment (production, staging, etc.)
   - Timestamp

## Step 4: Completion

**Purpose:** Register outputs, update pipeline state, and announce completion.

**Actions:**

1. Register outputs in `manifest.json` under `artifacts.5_deploy.outputs`:

```json
[
  {
    "type": "deploy-log",
    "path": ".lifecycle/features/{feature-name}/deploy-log.md",
    "created_at": "{ISO-8601}",
    "status": "complete"
  },
  {
    "type": "deploy-artifact",
    "path": "{deploy-url-or-artifact-path}",
    "created_at": "{ISO-8601}",
    "status": "complete"
  }
]
```

2. Set `artifacts.5_deploy.completed_at` = current ISO 8601 timestamp

3. Update `state.json`:
   - `current.status` = `"completed"`
   - `session.last_active` = current ISO 8601 timestamp
   - `session.resume_hint` = `"DEPLOY complete. Ready for DEPLOY TEST stage."`

4. Write history entry to `.lifecycle/history/`:

```json
{
  "timestamp": "{ISO-8601}",
  "from_stage": 4,
  "to_stage": 5,
  "gate_result": "passed",
  "missing_artifacts": [],
  "mode": "{current mode}"
}
```

5. Regenerate `.lifecycle/LIVING-STATE.md` with current state (per stage-transitions.md Step 6 procedure)

6. Announce completion:

```
DEPLOY Stage -- {feature-name} -- COMPLETE
==========================================
Deploy URL: {url or N/A}
Version: {version or N/A}
Platform: {platform}

Ready for DEPLOY TEST stage (smoke tests).
```

## Anti-Patterns

These are the most common mistakes. Do NOT do any of these:

| # | Anti-Pattern | Why It Is Wrong |
|---|-------------|-----------------|
| 1 | **Running production deploy commands directly** | DEPLOY is user-guided. Claude provides checklists and tracks completion, but the user executes deploy commands. Production environments require human oversight. |
| 2 | **Skipping project type confirmation** | Even if auto-detected, the project type must be confirmed by the user on first deploy. Wrong type leads to wrong checklist and potential deploy failures. |
| 3 | **Forgetting to record deploy URL** | The deploy URL/version is needed by DEPLOY TEST (Stage 6) for smoke testing. Without it, the next stage has no target. |
| 4 | **Not offering skip in feature mode** | Feature mode allows skipping DEPLOY. Forcing deployment in feature mode disrupts the developer workflow. Always check mode and offer skip. |
| 5 | **Treating skip as failure** | A skipped stage registers a `"skipped"` artifact so the gate for the next stage passes cleanly. Skip is a valid pipeline path, not an error. |

## Relationship to Other Skills

- **dev-lifecycle:** Primary skill for Stage 5. This adapter orchestrates the DEPLOY Stage workflow, manages project type resolution, and tracks checklist completion.
- **project-detection.md:** Referenced for project type detection and confirmation flow. Provides marker-based auto-detection.
- **Git:** Not directly involved in Stage 5 deployment, but commit artifacts from Stage 4 inform what is being deployed.
- **GSD:** Not directly involved in Stage 5. GSD handles project-level planning, not deployment.

## File Locations

| File | Path | Purpose |
|------|------|---------|
| This adapter | `$CLAUDE_SKILL_DIR/references/deploy-stage.md` | DEPLOY Stage workflow instructions |
| Commit stage adapter | `$CLAUDE_SKILL_DIR/references/commit-stage.md` | Previous stage (gate 4_to_5 source) |
| Deploy test adapter | `$CLAUDE_SKILL_DIR/references/deploy-test-stage.md` | Next stage (consumes deploy URL) |
| Project detection | `$CLAUDE_SKILL_DIR/references/project-detection.md` | Project type detection markers |
| Stage transitions | `$CLAUDE_SKILL_DIR/references/stage-transitions.md` | Gate conditions (4_to_5, 5_to_6) |
| State template | `$CLAUDE_SKILL_DIR/templates/state.json` | State schema with stage/status fields |
| Manifest template | `$CLAUDE_SKILL_DIR/templates/manifest.json` | Artifact registry with gate rules |

**Runtime location:** `$CLAUDE_SKILL_DIR` = `~/.claude/skills/dev-lifecycle`
**Version-controlled location:** `skill/` directory in the project repository
