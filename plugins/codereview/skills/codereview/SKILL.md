---
name: codereview
metadata:
  version: 1.2.0
description: >
  Automated pre-PR code review. Diffs current branch against main, analyzes all
  changed files, and produces a structured report with severity-rated findings,
  test coverage assessment, and a final grade. Use this skill whenever the user
  asks for code review, pre-PR review, code analysis, quality check, or wants
  to review changes before merging — even if they don't say "codereview" explicitly.
  Triggers: "code review", "pre-PR review", "review my code", "quality check",
  "review changes", "codereview", "check my code", "analyze code", "PR review"
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty). Valid inputs:

- Empty: full review of all changed files
- Focus area: `security`, `performance`, `types`, `bugs`, `tests`
- File path or glob: review only matching changed files (e.g. `src/components/Quote*`)
- Key-value overrides: `baseDir=app/ fileExtensions=ts,js uiLibReducedRigor=true` (see `references/configuration.md`)

Read `references/configuration.md` for default values and override syntax.

## Goal

Perform a comprehensive, automated code review of all changes in the current branch compared to the base branch. Produce a structured Markdown report with severity-rated findings, test coverage assessment, and a final grade. This replaces manual pre-PR review.

This skill is **stack-agnostic**. Defaults target TypeScript/React but all values are configurable. Set `frameworkPatterns=dotnet` for C#/.NET projects (WPF, WinForms, ASP.NET, Console) — this activates .NET-specific checks and deactivates web-frontend rules. The universal checks (readability, complexity, error handling, secrets) apply to every stack.

---

## Operating Constraints

**STRICTLY READ-ONLY**: Do **not** modify, create, or delete any files. Do **not** run destructive commands. Do **not** run formatters, linters, or fixers. Output ONLY a structured analysis report in the conversation.

**No Code Rewrites**: This skill identifies issues and suggests fixes in the report — it does NOT apply them.

## Error Handling

- **Git command failures**: include the exact failing git command in the error message and stop immediately (do not continue to the next step).
- **File read failures**: skip the file and record it as `Could not analyze: {filename} ({reason})` in the final report.
- **Git diff / merge-base / binary file issues**: flag the affected file as `unanalyzable` and continue with the remaining files.
- **Context / memory / token exhaustion**: finish analyzing files already processed, then record `Analysis incomplete due to context limits. {n} files not analyzed.` and proceed to the final report.
- **Timeout**: if analysis is taking too long, prioritize CRITICAL/HIGH checks on remaining files, skip MEDIUM/LOW, and note the truncation in the report.
- **Parsing errors** (malformed source, non-UTF-8 content, etc.): flag the file as `unanalyzable` and continue.

Regardless of which failures occur, always produce a final report that lists:

1. All files successfully analyzed (with their findings).
2. All failures (skipped / unanalyzable files and the reasons).

## Execution Steps

### 1. Gather Git Context

Run these git commands to understand the branch state:

```bash
# Abort early if not inside a git repository
git rev-parse --is-inside-work-tree 2>/dev/null || { echo "Error: not a git repository."; exit 1; }

# Determine base branch: prefer BASE_BRANCH env var, then origin HEAD, then try main/master
BASE_BRANCH="${BASE_BRANCH:-$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')}"
if [ -z "$BASE_BRANCH" ]; then
  if git show-ref --verify --quiet refs/heads/main 2>/dev/null; then
    BASE_BRANCH="main"
  elif git show-ref --verify --quiet refs/heads/master 2>/dev/null; then
    BASE_BRANCH="master"
  else
    echo "Error: could not detect base branch (no main or master found)."; exit 1
  fi
fi

# Current branch name (fail clearly on detached HEAD)
BRANCH_NAME=$(git rev-parse --abbrev-ref HEAD 2>/dev/null)
if [ -z "$BRANCH_NAME" ] || [ "$BRANCH_NAME" = "HEAD" ]; then
  echo "Error: detached HEAD state — cannot determine current branch."; exit 1
fi

# Prevent reviewing a branch against itself
if [ "$BRANCH_NAME" = "$BASE_BRANCH" ]; then
  echo "Cannot review base branch ($BASE_BRANCH) against itself."; exit 1
fi

# Common ancestor with base branch
MERGE_BASE=$(git merge-base "$BASE_BRANCH" HEAD 2>/dev/null) || { echo "Error: could not find merge base with $BASE_BRANCH (missing remote?)"; exit 1; }

# Files changed (names only)
git diff "$MERGE_BASE"...HEAD --name-only

# Change statistics
git diff "$MERGE_BASE"...HEAD --stat

# Commit log for this branch
git log "$MERGE_BASE"..HEAD --oneline --no-decorate
```

Capture and store:

- `BASE_BRANCH`: configured or auto-detected base branch
- `BRANCH_NAME`: current branch
- `MERGE_BASE`: the ancestor commit hash
- `CHANGED_FILES`: list of changed file paths
- `DIFF_STAT`: insertions/deletions summary
- `COMMIT_LOG`: list of commits on this branch

If `CHANGED_FILES` is empty, output a short message: "No changes detected between this branch and `$BASE_BRANCH`." and stop.

### 2. Filter and Classify Files

**Exclude** these files from analysis (still list them in the report header as "Excluded"):

- `**/package-lock.json`, `**/yarn.lock`, `**/pnpm-lock.yaml`
- `**/node_modules/**`
- `**/dist/**`, `**/build/**`, `**/.next/**`
- `**/*.min.js`, `**/*.min.css`
- Binary files (images, fonts, etc.)
- `**/.claude/**` (command and skill files themselves)
- `**/.claude/skills/**`

**Additional exclusions by `frameworkPatterns`:**

- `dotnet`: `**/bin/**`, `**/obj/**`, `**/*.Designer.cs`, `**/*.g.cs`, `**/*.g.i.cs`, `**/AssemblyInfo.cs`, `**/*.user`, `**/*.suo`

**Classify** remaining files into categories using the Configuration values:

| Category | Pattern (uses config values) | Review Rigor |
|----------|-------------------------------|--------------|
| `CODE` | `{baseDir}/**/*.{fileExtensions}` — excluding `testFilePatterns` and `generatedDirs` | Full analysis |
| `UI_LIB` | files matching `generatedDirs` (e.g. `src/components/ui/**`) | Reduced rigor when `uiLibReducedRigor=true` — only flag CRITICAL/HIGH issues |
| `TESTS` | files matching `testFilePatterns` (e.g. `src/**/*.test.ts`, `src/test/**`) | Test quality analysis |
| `CONFIG` | files matching `configFilePatterns` (e.g. `*.config.*`, `tsconfig*`, `package.json`) | Config-specific checks only |
| `DOCS` | `*.md`, `*.txt` (excluding `.claude/`) | Skip analysis, note in header |
| `STYLES` | files matching `styleFilePatterns` (e.g. `**/*.css`) | Minimal review |

If more than 15 files are classified as `CODE` or `TESTS`, prioritize files with the most changes (by diff stat lines changed). Note any deprioritized files in the report.

### 3. Load Context for Analysis

For each file, load the appropriate context:

- **CODE / TESTS**: Read the full git diff for the file AND the complete current file content. If the file imports local modules that were also changed, note the relationship.
- **UI_LIB**: Read only the git diff (not full file content).
- **CONFIG**: Read the git diff only.

Use the `Read` tool for file contents and `Bash` with `git diff` for diffs.

When a `CODE` file imports types or functions from other changed files, note these cross-file dependencies for coherence analysis.

### 4. Assess Test Coverage

Map each `CODE` file to its corresponding test file(s) using the following priority order. Probe each candidate path in order and stop at the first level that yields **at least one existing file**:

| Priority | Candidate pattern (given source `{dir}/{Base}.{ext}`) |
|----------|-------------------------------------------------------|
| 1 (highest) | `{dir}/{Base}.test.{ext}` then `{dir}/{Base}.spec.{ext}` — same directory, same extension |
| 2 | `{dir}/{Base}.test.ts`, `{dir}/{Base}.test.tsx`, `{dir}/{Base}.test.js`, `{dir}/{Base}.test.jsx` — same directory, all extensions |
| 3 | `{dir}/__tests__/{Base}.test.{ext}`, `{dir}/__tests__/{Base}.spec.{ext}` — `__tests__` sibling folder |
| 4 | `{testRoot}/{dir}/{Base}.test.{ext}`, `{testRoot}/{dir}/{Base}.spec.{ext}` — project test root |
| 5 (lowest) | Any file in `CHANGED_FILES` (TESTS category) whose basename matches `{Base}` (case-insensitive) |

**Auto-detect `{testRoot}`**: Check in order:

1. If `vitest.config.*` or `vite.config.*` exists, look for `test.root` or `test.include` paths in the config
2. If `jest.config.*` or `package.json` contains `jest.roots` or `jest.testMatch`, derive the test root
3. Fall back to the first of `src/test/`, `tests/`, `test/`, or `__tests__/` that exists at repo root

**When `frameworkPatterns=dotnet`**, use these candidate patterns instead:

| Priority | Candidate pattern (given source `{dir}/{Base}.{ext}`) |
|----------|-------------------------------------------------------|
| 1 (highest) | `{ProjectName}.Tests/{Base}Tests.cs` — sibling test project |
| 2 | `{ProjectName}.Tests/**/{Base}Tests.cs` — test project subdirectory |
| 3 | `{dir}/{Base}Tests.cs` — same directory |
| 4 (lowest) | Any file in `CHANGED_FILES` (TESTS category) matching `{Base}` (case-insensitive) |

**Auto-detect `{testRoot}` for dotnet**: Look for `*.csproj` files containing `<PackageReference Include="xunit"` or `Microsoft.NET.Test.Sdk` or `NUnit` or `MSTest`. The directory of such `.csproj` is the test root.

**Resolution rules:**

- A **match** at a priority level means one or more candidate paths resolve to real files in the repository.
- Matching stops at the first level that produces at least one match; lower-priority levels are not checked.
- When multiple test files match (e.g. both `.test.ts` and `.test.tsx` exist), treat them as a **single logical match group** — any one of them being in `CHANGED_FILES` satisfies the WITH_TESTS condition.

Classify each `CODE` file using the matched group:

- **WITH_TESTS**: A matching test file group was found AND at least one file in the group was modified in this branch (i.e. appears in `CHANGED_FILES`).
- **STALE_TESTS**: A matching test file group was found but **none** of the files in the group were modified in this branch despite production code changes.
- **NO_TESTS**: No matching test file found at any priority level.

### 5. Zen Principles Analysis

Apply these 5 principles as analysis lenses to all `CODE` files (reduced rigor for `UI_LIB`):

#### 5.1 "Beautiful is better than ugly" & "Readability counts"

**Universal:**

- Non-semantic variable/function names (single letters, abbreviations, misleading names)
- Inconsistent formatting within the changed code
- Magic numbers or strings without named constants
- Excessively long functions (>50 lines of logic)
- Missing or misleading documentation on exported/public functions
  - `react|vue|angular|node`: JSDoc (`/** ... */`)
  - `dotnet`: XML documentation comments (`/// <summary>`)

#### 5.2 "Explicit is better than implicit"

**Universal:**

- Missing parameter validation at public API boundaries
- Implicit type conversions that could lose data

**When `frameworkPatterns` is `react|vue|angular|node` (web frontend):**

- Missing TypeScript types or using `any`
- Implicit return types on exported functions
- React component props without explicit interface/type
- `useEffect` with missing or incorrect dependency arrays
- Implicit boolean coercion that could mask bugs (e.g., `value && <Component />` where value could be `0`)

**When `frameworkPatterns=dotnet`:**

- `var` used where type is genuinely ambiguous (non-obvious inference)
- `dynamic` keyword without strong justification
- Implicit conversions that could lose data (e.g., `long` to `int`)

#### 5.3 "Simple is better than complex"

**Universal:**

- Over-engineered abstractions for simple problems
- Unnecessary indirection (wrapper functions that just forward calls)
- Single Responsibility Principle violations (class/component doing too much)
- Premature optimization without evidence of need (flag as LOW with note: "Consider profiling to confirm benefit before applying")

**When `frameworkPatterns` is `react|vue|angular|node`:**

- Custom hooks that could be replaced with simpler patterns

**When `frameworkPatterns=dotnet`:**

- Business logic in XAML code-behind instead of ViewModel (MVVM violation)
- Over-abstracted service interfaces with single implementation and no test seam justification

#### 5.4 "Flat is better than nested"

**Universal:**

- Arrow code (>3 levels of nesting)
- Missing guard clauses (early returns)

**When `frameworkPatterns` is `react|vue|angular|node`:**

- Deeply nested ternary operators in JSX
- Callback pyramids (nested `.then()` chains or nested callbacks)
- Complex conditional rendering that could be extracted

**When `frameworkPatterns=dotnet`:**

- Deeply nested `if/else` chains that could use pattern matching or guard clauses
- Complex LINQ chains (>3 chained operations) that would be clearer as separate steps

#### 5.5 "Errors should never pass silently"

**Universal:**

- Empty `catch` blocks or catch that only logs without re-throwing or returning error
- API/service calls without error feedback path
- Silent fallbacks that hide bugs

**When `frameworkPatterns` is `react|vue|angular|node`:**

- Unhandled Promise rejections
- Missing error boundaries for component trees

**When `frameworkPatterns=dotnet`:**

- `async void` methods outside of UI event handlers (unobservable exceptions) — flag as CRITICAL
- `catch (Exception) { }` without logging or re-throw — flag as HIGH
- Missing `try/finally` or `using` for `IDisposable` resources — flag as HIGH

### 6. Additional Detection Passes

#### 6.1 Bug Detection

**Universal:**

- Potential null access without checks
- Off-by-one errors in loops or array operations
- `async` functions that never `await` anything (likely missing await or unnecessary async)

**When `frameworkPatterns` is `react|vue|angular|node`:**

- `useEffect` dependency array mismatches (missing deps or unnecessary deps)
- Race conditions from stale closures or unmounted component updates
- Direct state mutation (modifying state objects/arrays without creating new references)
- Incorrect equality checks (`==` instead of `===`)

**When `frameworkPatterns=dotnet`:**

- `async void` (except UI event handlers) — unhandled exceptions crash the app
- `IDisposable` not disposed (missing `using` statement)
- Race conditions with `Task.Run` and shared mutable state
- `DateTime.Now` instead of `DateTime.UtcNow` in cross-timezone logic
- Deadlocks from `.Result` or `.Wait()` on async code in UI thread

#### 6.2 Security

**Universal:**

- Exposed secrets, API keys, tokens in code or config
- Hardcoded API URLs, service endpoints, or connection strings (should use environment variables or config)
- `eval()`, `new Function()`, or dynamic code execution
- Missing input validation/sanitization at system boundaries

**When `frameworkPatterns` is `react|vue|angular|node`:**

- XSS vectors: `dangerouslySetInnerHTML`, unescaped user input in DOM
- Sensitive data in localStorage without encryption

**When `frameworkPatterns=dotnet`:**

- SQL string concatenation (use parameterized queries or Entity Framework)
- `MessageBox.Show()` or `OpenFileDialog` in service/domain classes (UI leak into business layer)
- `System.Windows.Forms` using in non-UI classes
- Hardcoded file paths without `Path.Combine()`

#### 6.3 Performance

**Universal:**

- Large classes/components that should be split for maintainability or lazy loading
- Expensive computations without caching or memoization

**When `frameworkPatterns` is `react|vue|angular|node`:**

- Inline object/array/function creation in JSX props (new reference every render)
- Missing `key` props or using array index as `key` in dynamic lists
- **`React.memo` missing** — flag as **MEDIUM** only when the component is rendered inside a list or loop, or when prop identity changes are known to cause unnecessary child re-renders. Otherwise flag as **LOW** or omit.
- **`useCallback` missing** — flag as **MEDIUM** only when the callback is passed as a prop to a memoized child or used in a `useEffect` dependency array and its identity changes provably cause repeated effect execution. Otherwise flag as **LOW** or omit.
- **`useMemo` missing** — flag as **MEDIUM** only when the computation is demonstrably expensive (>100ms measured, or explicitly identified by profiling). For cheap computations flag as **LOW** or omit.
- **All other memoization suggestions** — assign **LOW** and include a note: _"Recommend running a profiler before applying this optimization to confirm a measurable benefit."_

**When `frameworkPatterns=dotnet`:**

- `new HttpClient()` per-call instead of `IHttpClientFactory` or injected instance
- String concatenation in loops (use `StringBuilder`)
- `Thread.Sleep()` in async code or tests
- LINQ `.ToList()` or `.ToArray()` when `IEnumerable` suffices (unnecessary materialization)

#### 6.4 Type Safety

**Universal:**

- Missing discriminated union / exhaustive switch checks (switch without default)
- Excessive type casting that bypasses type checking

**When `frameworkPatterns` is `react|vue|angular|node`:**

- Usage of `any` type (explicit or implicit)
- Type assertions (`as Type`) that bypass type checking
- Missing return types on exported functions
- Optional chaining chains longer than 3 levels (`a?.b?.c?.d?.e`)

**When `frameworkPatterns=dotnet`:**

- `public static` mutable fields used as service locator pattern
- Missing null checks on deserialized objects (JSON/XML)
- Casting with `(Type)obj` instead of pattern matching (`is Type t`)
- `object` or `dynamic` used where a generic `<T>` or interface fits

### 7. Assign Severities

Each finding gets one severity:

| Severity | Criteria | Action |
|----------|----------|--------|
| **CRITICAL** | Security vulnerabilities, data loss risk, crashes in production, exposed secrets | Must fix before merge |
| **HIGH** | Bugs likely to manifest, missing error handling on user-facing flows, `any` on public API | Should fix before merge |
| **MEDIUM** | Code smell, minor Zen violations, missing tests for new logic, performance concerns | Recommend fixing |
| **LOW** | Style preferences, minor readability improvements, suggestions for future improvement | Optional improvement |

### 8. Produce Structured Report

Read `references/report-template.md` for the full report structure, grading scale, and examples.

Output the Markdown report directly in the conversation following that template.

### 9. Handle Special Cases

- **Zero findings**: Output a congratulatory report. Grade A across all criteria. Still show the header, test coverage table, and grade table.
- **$ARGUMENTS matches a focus area**: Only run the matching detection passes from steps 5-6. Still show full report structure but mark non-analyzed sections as "Not analyzed (focused review on {area})".
- **$ARGUMENTS matches a file path/glob**: Only analyze matching files from the changed files list. Show only those files in the report.
- **UI_LIB files**: Apply only CRITICAL and HIGH severity checks. Note in findings: "(UI_LIB - reduced rigor)".
- **More than 50 findings**: Show all CRITICAL/HIGH/MEDIUM findings first, then as many LOW as fit within the 50-finding cap. Add overflow count. Recommend running focused reviews per file.

## Operating Principles

### Context Efficiency

- **Load diffs before full files**: Only read complete file content when the diff suggests deeper analysis is needed
- **Prioritize by change size**: Files with more changes get more thorough analysis
- **Cap analysis scope**: Maximum 15 full file reads to avoid context exhaustion
- **Be specific**: Every finding must reference a specific file and line number from the actual diff

### Analysis Integrity

- **NEVER modify files** — this is strictly read-only analysis
- **NEVER hallucinate line numbers** — only reference lines you actually read from the diff or file
- **NEVER invent findings** — if the code is clean, say so. A clean report is a valid outcome.
- **Be fair to generated code** — UI_LIB files get reduced scrutiny
- **Acknowledge context limits** — if you couldn't fully analyze a file, say so in the report
- **Ground findings in evidence** — quote the problematic code snippet in the finding description when helpful
