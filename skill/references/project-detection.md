# Project Type Detection

This document defines how dev-lifecycle automatically detects the project type by scanning for marker files in the project root. The detected type determines which stage-specific templates are used (deploy checklists, test commands, build scripts, etc.).

## Detection Markers

| Marker File | Project Type | Sub-type Detection |
|-------------|-------------|-------------------|
| `package.json` | `node-web` | Check for framework: `next` (Next.js), `react` (React SPA), `vue` (Vue), `angular` (Angular), `express` (Express API), `fastify` (Fastify API) |
| `pubspec.yaml` | `flutter` | Check for `flutter` key in pubspec |
| `Cargo.toml` | `rust` | Check for `[[bin]]` (CLI) vs `[lib]` (library) |
| `go.mod` | `go` | Check for `main` package (CLI/API) |
| `requirements.txt` or `pyproject.toml` | `python` | Check for framework: `django`, `flask`, `fastapi`, `click` (CLI) |
| `*.xcodeproj` or `*.xcworkspace` | `ios` | Native iOS / macOS app |
| `build.gradle` or `build.gradle.kts` | `android-jvm` | Android or JVM project |
| `Makefile` (alone, no other markers) | `c-cpp` | C or C++ project |
| `*.sln` or `*.csproj` | `dotnet` | .NET project |

## Detection Flow

1. **Scan project root** for marker files listed above
2. **Determine type** -- match markers to project types. A project may match multiple types (see Multi-Type Projects below)
3. **Present to user for confirmation:**
   ```
   Detected project type: [type]
   Is this correct? (yes/no/specify)
   ```
4. **Store in state.json:**
   - `project.detected_type` -- the detected type string (or array for multi-type)
   - `project.confirmed` -- set to `true` after user confirms
5. **Use type for stage-specific templates** -- deploy checklists, test commands, build scripts are loaded based on confirmed project type

## Detection Priority

When multiple markers are found, apply this priority:

1. Framework-specific markers take precedence over generic ones (e.g., `pubspec.yaml` wins over a co-located `Makefile`)
2. If two equal-priority markers exist, treat as multi-type project
3. If no markers found, prompt user to specify manually

## Multi-Type Projects

Projects can have multiple types. Common examples:

| Combination | Example |
|-------------|---------|
| `flutter` + `python` | Flutter frontend with Python (FastAPI) backend |
| `node-web` + `python` | React frontend with Django backend |
| `go` + `node-web` | Go API with React admin panel |
| `rust` + `python` | Rust core with Python bindings |

For multi-type projects:
- Store `detected_type` as an array: `["flutter", "python"]`
- Present all detected types to user for confirmation
- Stage templates may differ per component (e.g., deploy Stage 5 has separate steps for frontend and backend)
- User can remove false positives during confirmation

## Sub-type Detection Details

### Node.js (`package.json`)

Scan `dependencies` and `devDependencies` for:

| Package | Sub-type |
|---------|----------|
| `next` | `node-web/next` |
| `react` (without next) | `node-web/react` |
| `vue` | `node-web/vue` |
| `@angular/core` | `node-web/angular` |
| `express` | `node-web/express` |
| `fastify` | `node-web/fastify` |
| None of above | `node-web/generic` |

### Python (`requirements.txt` or `pyproject.toml`)

Scan dependencies for:

| Package | Sub-type |
|---------|----------|
| `django` | `python/django` |
| `flask` | `python/flask` |
| `fastapi` | `python/fastapi` |
| `click` or `typer` | `python/cli` |
| None of above | `python/generic` |

## When Detection Runs

- **First invocation** in a new project (no `.lifecycle/state.json` exists)
- **User requests** re-detection (`/dev-lifecycle detect`)
- **Never automatically** after initial confirmation -- respects user's choice
