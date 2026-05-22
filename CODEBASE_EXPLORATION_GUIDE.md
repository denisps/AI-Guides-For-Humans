# How to Quickly Explore and Understand Any Codebase

### A Practical Guide for Software Developers

---

## Table of Contents

1. [The Explorer's Mindset](#1-the-explorers-mindset)
2. [Phase 1 — Orient Yourself in 5 Minutes](#2-phase-1--orient-yourself-in-5-minutes)
3. [Phase 2 — Read the Map Before You Walk](#3-phase-2--read-the-map-before-you-walk)
4. [Phase 3 — Find the Entry Points](#4-phase-3--find-the-entry-points)
5. [Phase 4 — Search Like a Detective](#5-phase-4--search-like-a-detective)
6. [Phase 5 — Follow the Data Flow](#6-phase-5--follow-the-data-flow)
7. [Phase 6 — Understand the Dependencies](#7-phase-6--understand-the-dependencies)
8. [Phase 7 — Mine the Git History](#8-phase-7--mine-the-git-history)
9. [Phase 8 — Run It and Break It](#9-phase-8--run-it-and-break-it)
10. [Phase 9 — Build Your Mental Model](#10-phase-9--build-your-mental-model)
11. [Language-Specific Cheat Sheets](#11-language-specific-cheat-sheets)
12. [Tools Reference](#12-tools-reference)
13. [Common Patterns Across Codebases](#13-common-patterns-across-codebases)

---

## 1. The Explorer's Mindset

Before opening a single file, understand what kind of explorer you need to be. A large codebase is not a novel you read from page 1 to the end — it is a city. You do not explore a city by walking every street in order. You find landmarks, read the map, follow signs, and ask locals.

**Core principles to internalize:**

- **Breadth first, depth second.** Understand the shape of the whole before diving into any part.
- **Follow behavior, not files.** Ask "what happens when X occurs?" and trace that path.
- **Trust the names.** Good code is self-documenting. A function named `parse_config_file` almost certainly parses a config file. Read names before reading bodies.
- **Tests are documentation.** A test suite describes what the code is *supposed* to do, often more honestly than comments.
- **You do not need to understand everything.** You only need to understand the part relevant to your current task.

---

## 2. Phase 1 — Orient Yourself in 5 Minutes

### Step 1: Get a High-Level Directory Listing

The first command you run should always be a top-level directory listing.

```bash
# List top-level contents with sizes and types
ls -la

# On macOS/Linux, get a tree view (install if needed: brew install tree / apt install tree)
tree -L 2 --dirsfirst

# On Windows (PowerShell)
Get-ChildItem -Recurse -Depth 2 | Select-Object FullName
```

**What to look for:**

| Directory/File Pattern | What It Tells You |
|---|---|
| `src/`, `lib/`, `app/` | Where the main source code lives |
| `tests/`, `test/`, `spec/`, `__tests__/` | Where tests live |
| `docs/`, `doc/` | Project documentation |
| `scripts/`, `tools/`, `bin/` | Automation and CLI utilities |
| `config/`, `configs/`, `conf/` | Configuration files |
| `Makefile`, `CMakeLists.txt` | C/C++ build system |
| `package.json`, `yarn.lock`, `pnpm-lock.yaml` | Node.js project |
| `pyproject.toml`, `setup.py`, `requirements.txt` | Python project |
| `Cargo.toml` | Rust project |
| `go.mod` | Go project |
| `pom.xml`, `build.gradle` | Java/JVM project |
| `.github/workflows/` | CI/CD pipeline definitions |
| `docker-compose.yml`, `Dockerfile` | Containerized deployment |

### Step 2: Count the Files

Understanding scale prevents you from underestimating the effort.

```bash
# Count all source files by extension
find . -type f | sed 's/.*\.//' | sort | uniq -c | sort -rn | head -20

# Count total lines of code (requires cloc: brew install cloc / apt install cloc)
cloc .

# A quick approximation with wc
find . -name "*.py" | xargs wc -l | tail -1
find . -name "*.ts" | xargs wc -l | tail -1
```

**Example output from `cloc`:**
```
-------------------------------------------------------------------------------
Language                     files          blank        comment           code
-------------------------------------------------------------------------------
Python                         312           8201           6432          42103
YAML                            48            512            201           3821
Markdown                        22           1203              0           4102
...
```

This tells you instantly: this is primarily a Python project with ~42K lines.

### Step 3: Spot the Language and Framework

```bash
# Check what languages are in play
find . -type f \( -name "*.py" -o -name "*.ts" -o -name "*.go" -o -name "*.rs" \) \
  | sed 's/.*\.//' | sort | uniq -c | sort -rn

# Identify the framework from config files
cat package.json 2>/dev/null | grep -E '"(react|vue|angular|next|express|fastify)"'
cat pyproject.toml 2>/dev/null | grep -E '(django|flask|fastapi|torch|tensorflow)'
cat Cargo.toml 2>/dev/null | grep -E '(tokio|actix|axum|rocket)'
```

---

## 3. Phase 2 — Read the Map Before You Walk

### Read These Files First (in order)

Almost every professional project has at least some of these. Read them before touching any source file.

```bash
# 1. The main README
cat README.md

# 2. Any contributing guide — often contains architecture notes
cat CONTRIBUTING.md
cat docs/contributing.md

# 3. A changelog — tells you the project's history in compressed form
cat CHANGELOG.md
cat HISTORY.md

# 4. The license — identifies the project's community and constraints
cat LICENSE

# 5. Architecture or design documents
ls docs/
find docs/ -name "*.md" | head -20
```

### Scan the CI/CD Pipeline

The CI pipeline is a goldmine — it shows you exactly how to build, test, and lint the project.

```bash
# GitHub Actions
cat .github/workflows/*.yml

# GitLab CI
cat .gitlab-ci.yml

# CircleCI
cat .circleci/config.yml

# Travis CI
cat .travis.yml
```

**Example `.github/workflows/ci.yml` snippet:**
```yaml
jobs:
  test:
    steps:
      - run: pip install -e ".[dev]"   # <- how to install
      - run: pytest tests/             # <- how to test
      - run: ruff check .              # <- linter used
      - run: mypy src/                 # <- type checker used
```

In 10 lines of YAML, you learned the build tool, test framework, linter, and type checker.

### Read the Makefile or Task Runner

```bash
# See all available make targets
make help 2>/dev/null || grep -E '^[a-zA-Z_-]+:.*?##' Makefile | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

# npm scripts
cat package.json | python3 -m json.tool | grep -A 30 '"scripts"'

# Python task runner (Taskfile, invoke, etc.)
cat Taskfile.yml 2>/dev/null
cat tasks.py 2>/dev/null
```

---

## 4. Phase 3 — Find the Entry Points

The **entry point** is where execution begins. Everything else flows from here. Finding it is your most important early task.

### Web Servers / APIs

```bash
# Python: look for app creation, server start, main guard
grep -rn "if __name__" --include="*.py" . | head -20
grep -rn "app = Flask\|app = FastAPI\|app.run\|uvicorn.run" --include="*.py" . | head -20

# Node.js: main field in package.json
cat package.json | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('main',''))"

# Look for server/index files
find . -name "index.js" -o -name "server.js" -o -name "main.ts" -o -name "app.ts" | head -10

# Go: main package
find . -name "main.go" | head -10
```

### CLI Tools

```bash
# Python: console_scripts in setup.py / pyproject.toml
grep -A 5 "console_scripts" setup.py pyproject.toml 2>/dev/null

# Look for argparse / click / typer
grep -rn "argparse\|click\|typer\|fire" --include="*.py" . | grep "import" | head -10

# Node.js: bin field
cat package.json | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('bin',''))"
```

### Libraries / Packages

For libraries, the entry point is the public API surface — the `__init__.py` or `index.ts`.

```bash
# Python: the top-level __init__.py is the public API
cat src/mypackage/__init__.py

# TypeScript: the index file
cat src/index.ts
cat src/index.d.ts  # type declarations

# Rust: lib.rs or main.rs
cat src/lib.rs | head -60
cat src/main.rs | head -40
```

**Pro tip:** In Python, anything imported in `__init__.py` is public. Anything not imported is internal. This tells you what the package authors consider important.

---

## 5. Phase 4 — Search Like a Detective

Searching is your core skill. Master it and you can navigate any codebase.

### grep — Your Primary Tool

```bash
# Basic: find all uses of a function
grep -rn "function_name" --include="*.py" .

# Case-insensitive search
grep -rni "config" --include="*.py" . | head -20

# Show context lines around match (-A after, -B before, -C both)
grep -rn "def authenticate" --include="*.py" . -A 10

# Search for a pattern only in test files
grep -rn "assert" --include="*test*.py" . | wc -l

# Find all TODO/FIXME comments (useful for understanding known issues)
grep -rn "TODO\|FIXME\|HACK\|XXX\|NOTE" --include="*.py" . | head -40

# Find all places a class is instantiated
grep -rn "MyClass(" --include="*.py" .

# Exclude directories (e.g., node_modules, .git)
grep -rn "pattern" --include="*.js" --exclude-dir={node_modules,.git,dist} .
```

### ripgrep (rg) — grep on steroids

If you can install one tool, install `ripgrep`. It is dramatically faster than `grep` and `.gitignore`-aware.

```bash
# Install
brew install ripgrep      # macOS
apt install ripgrep       # Debian/Ubuntu
winget install BurntSushi.ripgrep  # Windows

# Basic usage (same as grep but faster, prettier output)
rg "function_name" --type py

# Search for a string in a specific file type
rg "import torch" -t py

# Search with file name only (no content)
rg -l "TODO" -t py

# Count matches per file
rg -c "assert" -t py | sort -t: -k2 -rn | head -20

# Show only the matching part (not the whole line)
rg -o "class \w+" -t py | sed 's/.*://' | sort -u

# Search for a regex pattern
rg "def (get|set|update)_\w+" -t py
```

### find — Locate Files by Name or Property

```bash
# Find all Python files modified in the last 7 days
find . -name "*.py" -mtime -7

# Find files by name pattern
find . -name "*config*" -type f
find . -name "test_*.py" -type f

# Find large files (sometimes reveals generated/binary artifacts)
find . -size +1M -type f | sort

# Find files containing a specific string (older method, grep is better)
find . -name "*.py" -exec grep -l "MyClass" {} \;

# Find all __init__.py files (Python package structure)
find . -name "__init__.py" | sort
```

### Searching for Definitions

```bash
# Find where a Python class is DEFINED (not imported or used)
grep -rn "^class MyClass" --include="*.py" .

# Find where a function is DEFINED
grep -rn "^def my_function\|^    def my_function" --include="*.py" .

# Find all class definitions in a project
grep -rn "^class " --include="*.py" . | sed 's/:.*$//' | sort

# TypeScript/JavaScript: find where a function is exported
grep -rn "export function\|export const\|export class\|export default" --include="*.ts" . | head -30

# Go: find all exported function signatures
grep -rn "^func [A-Z]" --include="*.go" . | head -30
```

---

## 6. Phase 5 — Follow the Data Flow

Understanding how data flows through the system is more valuable than memorizing file contents. Pick a concrete scenario and trace it end-to-end.

### Technique: The "What Happens When..." Trace

Pick the most central operation the code performs and trace it through.

**Example: "What happens when a user logs in?"**

```bash
# Step 1: Find the login route/handler
rg -n "login" --type py | grep -i "route\|endpoint\|handler\|view"

# Step 2: Read that function and identify what it calls
# (e.g., it calls authenticate_user())
rg -n "def authenticate_user" --type py

# Step 3: Follow authenticate_user — what does it call?
# (e.g., it calls UserRepository.find_by_email())
rg -n "def find_by_email" --type py

# Step 4: Continue until you reach a database call or external service
rg -n "session.query\|db.execute\|cursor.execute" --type py
```

Write down the chain as you go:

```
POST /login
  → auth/views.py: login_view()
    → auth/services.py: authenticate_user()
      → users/repository.py: UserRepository.find_by_email()
        → db/session.py: execute SQL
      → auth/tokens.py: generate_jwt()
  → returns 200 + token
```

This simple diagram is worth hours of reading random files.

### Technique: Trace an Error Message

Error messages are incredibly useful navigation anchors.

```bash
# If you see error "Invalid configuration key: foo", find where it's raised
rg -rn "Invalid configuration key" .

# Find the function that raises it, then trace backwards from there
```

### Technique: Find All Callers of a Function

```bash
# Who calls my_important_function?
rg -rn "my_important_function(" --type py

# Who imports a specific module?
rg -rn "from mymodule import\|import mymodule" --type py

# Who uses a specific class?
rg -rn "MyClass\b" --type py | grep -v "class MyClass\|# "
```

---

## 7. Phase 6 — Understand the Dependencies

### Read the Dependency Manifest

```bash
# Python
cat requirements.txt
cat pyproject.toml
cat setup.py | grep -A 20 "install_requires"

# Node.js
cat package.json | python3 -m json.tool

# Rust
cat Cargo.toml

# Go
cat go.mod

# Ruby
cat Gemfile

# Java (Maven)
cat pom.xml | grep -A 3 "<dependency>"
```

### Understand Why a Dependency Exists

When you see an unfamiliar library, a quick search tells you its role:

```bash
# Find all imports of a library
rg -rn "import redis\|from redis" --type py

# See how it's used — what methods are called
rg -rn "redis\." --type py | head -30
```

### Spot Internal Module Dependencies

```bash
# Python: find all internal imports (tells you which modules depend on which)
rg -rn "^from \." --type py | awk -F: '{print $1}' | sort | uniq -c | sort -rn

# Which modules are most imported internally? (high number = core module)
grep -rh "^from \." --include="*.py" . | sed 's/from \.//' | sed 's/ import.*//' | \
  sort | uniq -c | sort -rn | head -20
```

A module that is imported by many others is a **core module** — understand it well.

### Visualize the Module Graph (Optional)

```bash
# Python: pydeps creates a visual dependency graph
pip install pydeps
pydeps mypackage --max-bacon=5 --show-deps

# Or use pyreverse (from pylint)
pip install pylint
pyreverse -o png -p MyProject src/mypackage/
```

---

## 8. Phase 7 — Mine the Git History

The git log is a narrative of the codebase. It tells you *why* things are the way they are.

### Start with a High-Level Log

```bash
# Compact one-line log
git log --oneline -30

# Log with graph (shows branching)
git log --oneline --graph --all -30

# Log with stats (shows which files changed)
git log --stat -10

# Search commit messages for a feature or bug
git log --oneline --grep="authentication"
git log --oneline --grep="fix\|bug" -i | head -20
```

### Find When and Why a Line Was Written

```bash
# Who wrote each line of a file, and in which commit?
git blame src/auth/views.py

# Show blame with commit date
git blame --date=short src/auth/views.py

# Find the commit that introduced a specific string
git log -S "def authenticate_user" --oneline

# Show the full diff of a specific commit
git show abc1234

# Show what changed in a file over time
git log --follow -p src/auth/views.py | head -100
```

### Find the Most Changed Files

Frequently changed files are often the most important (or most buggy).

```bash
# Top 20 most frequently modified files
git log --pretty=format: --name-only | grep -v '^$' | sort | uniq -c | sort -rn | head -20
```

### Explore Major Features via Branches / Tags

```bash
# List all tags (releases)
git tag --sort=-version:refname | head -20

# Compare what changed between two releases
git diff v1.0.0..v2.0.0 --stat

# Find feature branches
git branch -a | grep -i feature
```

---

## 9. Phase 8 — Run It and Break It

Reading code gives you a static picture. Running it gives you a dynamic one. Both are necessary.

### Get It Running

```bash
# Read the README install instructions first, then:

# Python
python -m venv .venv
source .venv/bin/activate  # Linux/macOS
.\.venv\Scripts\Activate.ps1  # Windows PowerShell
pip install -e ".[dev]"   # editable install with dev deps

# Node.js
npm install
npm run dev

# Go
go run ./cmd/server/

# Rust
cargo run
```

### Run the Tests

Tests are executable documentation. Run them and read them.

```bash
# Python
pytest -v                          # all tests, verbose
pytest tests/test_auth.py -v       # specific file
pytest -k "login" -v               # tests matching a name pattern
pytest --co -q                     # collect only — see all test names without running

# Node.js
npm test
npx jest --listTests               # list all test files

# Go
go test ./...                      # all packages
go test ./auth/... -v              # specific package, verbose

# Rust
cargo test
cargo test -- --list               # list all tests
```

### Add Debug Logging / Print Statements

When tracing a flow you don't understand, temporary print statements are your friend.

```bash
# Python: add a breakpoint or print
# In the source file, add:
# import pdb; pdb.set_trace()   # interactive debugger
# print(f"[DEBUG] variable={variable!r}")

# Run with verbose output
python -u script.py  # -u for unbuffered output

# See all print output even in pytest
pytest -s -v tests/test_auth.py
```

### Read Logs

```bash
# If the app writes logs to a file
tail -f app.log
tail -f logs/*.log

# On Linux, check systemd journal
journalctl -u myservice -f

# Docker logs
docker logs -f container_name
docker-compose logs -f service_name
```

---

## 10. Phase 9 — Build Your Mental Model

After the first several hours of exploration, stop and consolidate what you've learned.

### Draw a Component Diagram

Even a rough hand-drawn or text-based diagram is valuable. Use ASCII art or any tool you like.

```
                    ┌──────────────┐
     HTTP Request   │              │
  ───────────────►  │  API Layer   │
                    │  (FastAPI)   │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │   Service    │
                    │   Layer      │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
       ┌──────▼───┐  ┌─────▼────┐  ┌───▼──────┐
       │  Database│  │  Cache   │  │  External│
       │  (SQLite)│  │  (Redis) │  │  API     │
       └──────────┘  └──────────┘  └──────────┘
```

### Identify the Core Abstractions

Every codebase has 3–10 core abstractions (classes, types, or concepts) that everything else is built around. Finding them unlocks understanding.

```bash
# Find the most-referenced classes
rg -roh "class \w+" --type py | sed 's/class //' | sort | uniq -c | sort -rn | head -20

# Find the most-imported internal modules
grep -rh "^from \." --include="*.py" . | grep import | \
  sed 's/from \.\([a-z_]*\).*/\1/' | sort | uniq -c | sort -rn | head -10
```

### Write a One-Page Summary

Force yourself to write (even just in a scratch file) a one-page summary of:

1. What does this codebase do?
2. What are the 5 most important files?
3. How does data flow through the system for the most common operation?
4. What are the main abstractions and their responsibilities?
5. What parts do you still not understand?

This act of writing reveals gaps in your understanding and cements what you know.

---

## 11. Language-Specific Cheat Sheets

### Python

```bash
# Find all classes and their inheritance
grep -rn "^class " --include="*.py" . | sed 's/://'

# Find all async functions (useful in async codebases)
grep -rn "^async def\|^    async def" --include="*.py" . | head -30

# Find all decorators used (shows frameworks and patterns)
grep -rn "^@\|^    @" --include="*.py" . | grep -v "@pytest\|@property" | \
  sed 's/.*@//' | sed 's/(.*//' | sort | uniq -c | sort -rn | head -20

# Show the public interface of a module
python3 -c "import mymodule; help(mymodule)"
python3 -c "import mymodule; print(dir(mymodule))"

# Show source of a function interactively
python3 -c "import inspect, mymodule; print(inspect.getsource(mymodule.my_function))"
```

### JavaScript / TypeScript

```bash
# Find all exported functions/classes/types
rg -n "^export (function|class|const|type|interface|enum)" --type ts | head -40

# Find all React components
rg -n "export default function|export const \w+ = \(.*\) =>" --type tsx | head -30

# Find all API routes (Express)
rg -n "router\.(get|post|put|delete|patch)\|app\.(get|post|put|delete)" --type js | head -30

# Find all environment variable usages
rg -n "process\.env\." --type ts | grep -o "process\.env\.\w*" | sort -u

# List all npm scripts
cat package.json | node -e "const d=require('/dev/stdin'); console.log(Object.keys(d.scripts||{}).join('\n'))"
```

### Go

```bash
# Find all exported functions
grep -rn "^func [A-Z]" --include="*.go" . | head -30

# Find all struct definitions
grep -rn "^type \w* struct" --include="*.go" . | head -30

# Find all interface definitions
grep -rn "^type \w* interface" --include="*.go" . | head -30

# List all packages in the module
find . -name "*.go" -exec dirname {} \; | sort -u | grep -v vendor

# Find main entry point
find . -name "main.go" | xargs grep -l "func main()"
```

### Rust

```bash
# Find all public structs and enums
grep -rn "^pub struct\|^pub enum" --include="*.rs" . | head -30

# Find all trait implementations
grep -rn "^impl " --include="*.rs" . | head -30

# Find all public functions
grep -rn "^pub fn\|^    pub fn" --include="*.rs" . | head -30

# Show the crate's public API (from the docs)
cargo doc --open
```

### Java / Kotlin

```bash
# Find all class definitions
grep -rn "^public class\|^public abstract class\|^public interface" --include="*.java" . | head -30

# Find all Spring controllers/services (Spring Boot)
grep -rn "@Controller\|@RestController\|@Service\|@Repository" --include="*.java" . | head -30

# Find all API endpoints
grep -rn "@GetMapping\|@PostMapping\|@RequestMapping" --include="*.java" . | head -30
```

---

## 12. Tools Reference

### Essential Tools (install these first)

| Tool | Purpose | Install |
|---|---|---|
| `ripgrep` (`rg`) | Fast regex search | `brew install ripgrep` |
| `fd` | Fast file finder (replaces `find`) | `brew install fd` |
| `bat` | Cat with syntax highlighting | `brew install bat` |
| `tree` | Visual directory tree | `brew install tree` |
| `cloc` | Count lines of code | `brew install cloc` |
| `jq` | Parse JSON in terminal | `brew install jq` |
| `delta` | Beautiful git diffs | `brew install git-delta` |

### Editor Tools

Your editor is your most powerful exploration tool. Learn these shortcuts deeply:

**VS Code:**
- `Ctrl+P` / `Cmd+P` — Fuzzy file finder (fastest way to open any file)
- `Ctrl+Shift+F` / `Cmd+Shift+F` — Search across all files
- `F12` — Go to definition
- `Shift+F12` — Find all references
- `Alt+F12` — Peek definition (inline, without leaving current file)
- `Ctrl+G` — Go to line number
- `Ctrl+Shift+O` / `Cmd+Shift+O` — Jump to symbol in file
- `Ctrl+T` / `Cmd+T` — Jump to symbol across workspace
- `Ctrl+Click` / `Cmd+Click` — Go to definition of any symbol

**Vim/Neovim:**
- `gd` — Go to definition (with LSP)
- `gr` — Go to references (with LSP)
- `:grep <pattern> **/*.py` — Search across files
- `:e %:h/<Tab>` — Explore directory of current file

### Advanced: `ctags` and Language Servers

For large codebases without LSP support:

```bash
# Generate a tags file for vim/emacs
ctags -R --languages=Python .

# Jump to definition in vim: place cursor on symbol, press Ctrl+]
# Jump back: Ctrl+T
```

---

## 13. Common Patterns Across Codebases

Knowing common architectural patterns lets you recognize structure immediately.

### MVC (Model-View-Controller)

```
app/
  models/       ← data structures, database logic
  views/        ← templates, response formatting
  controllers/  ← request handling, business logic
```

```bash
# Quickly identify MVC by finding these directories
find . -type d \( -name models -o -name views -o -name controllers \)
```

### Layered Architecture (Onion / Hexagonal)

```
src/
  domain/       ← pure business logic, no dependencies
  application/  ← use cases, orchestrates domain
  infrastructure/ ← databases, external APIs, frameworks
  api/          ← HTTP handlers, serialization
```

### Feature-Based Structure

```
src/
  auth/
    auth.controller.ts
    auth.service.ts
    auth.model.ts
    auth.test.ts
  users/
    users.controller.ts
    ...
```

```bash
# Identify feature-based structure
find src -name "*.controller.*" | sed 's|/[^/]*$||' | sort -u
```

### The Worker/Queue Pattern

```bash
# Find queue/worker patterns
rg -rn "celery\|rq\|bull\|sidekiq\|rabbitmq\|kafka" . | grep "import\|require" | head -20
rg -rn "@task\|@job\|queue\.enqueue\|delay()" --type py | head -20
```

---

## Putting It All Together: A 90-Minute Protocol

Use this sequence when dropped into an unfamiliar codebase with a task to complete.

| Time | Activity |
|---|---|
| 0–5 min | `ls`, `tree -L 2`, `cloc .` — understand scale and language |
| 5–15 min | Read `README.md`, `CONTRIBUTING.md`, CI config |
| 15–25 min | Find entry points, read `__init__.py` or `index.ts` |
| 25–40 min | Run the test suite, read test names to understand behavior |
| 40–55 min | Search for your specific task area with `rg` |
| 55–70 min | Trace the data flow for the relevant operation |
| 70–80 min | Check git blame/log for the relevant files |
| 80–90 min | Write a one-paragraph summary of what you've learned |

After 90 minutes, you should be able to:
- Name the 3–5 most important files for your task
- Describe the data flow at a high level
- Know where to make a change and why

---

## Final Tips

1. **When lost, find a test.** Tests call real code with real arguments. Reading a test that covers your area is the fastest way to understand how something works.

2. **When searching, be specific first, then broad.** Start with the most specific search term you have. If that fails, broaden.

3. **Comment as you go.** Write comments in a scratch file as you explore. The act of writing forces clarity.

4. **Pair the static with the dynamic.** Read the code, then run it, then read it again. The second reading always reveals things the first missed.

5. **Respect the naming conventions.** If the project uses `_service.py` for service files, search for `*_service.py` to find all services. Conventions are breadcrumbs.

6. **Ask the git log before asking a human.** Most "why was this done this way?" questions are answered in a commit message.

7. **There is no shortcut to deep understanding — but there are shortcuts to working understanding.** You do not need to understand 100% of the code to make progress. Understand what you need, make progress, then come back and deepen.

---

*Good luck. The codebase you are exploring was built by humans with limited time, competing priorities, and changing requirements — just like you. Read it with generosity.*
