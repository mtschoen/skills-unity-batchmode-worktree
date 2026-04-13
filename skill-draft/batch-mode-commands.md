# Unity Batch Mode Command Templates

Copy-paste reference for running Unity in `-batchmode` from your worktree. All output paths assume the gitignored `TestResults/` directory at your worktree root.

## Variables

```bash
UNITY="C:/Program Files/Unity/Hub/Editor/<Version>/Editor/Unity.exe"
PROJECT="<your-worktree>/<project>"          # Unity project root
RESULTS="<your-worktree>/TestResults"        # gitignored artifact dir
STAMP="$(date +%Y%m%d-%H%M%S)"
```

## Compile check

Exits after import/compile. Non-zero exit on compile error. Expected runtime: 30–120s cold.

```bash
"$UNITY" -batchmode -nographics -quit \
  -projectPath "$PROJECT" \
  -logFile "$RESULTS/compile-$STAMP.log"
```

## EditMode tests

Runs Unity Test Framework EditMode tests. Expected runtime: 60–180s.

```bash
"$UNITY" -batchmode -nographics \
  -projectPath "$PROJECT" \
  -runTests -testPlatform EditMode \
  -testResults "$RESULTS/editmode-$STAMP.xml" \
  -logFile "$RESULTS/editmode-$STAMP.log"
```

## PlayMode tests

Same pattern, `-testPlatform PlayMode`. Expected runtime: 120–300s (cold reimport dominates). Needs a graphics-capable context on some packages — drop `-nographics` if you hit rendering errors.

```bash
"$UNITY" -batchmode \
  -projectPath "$PROJECT" \
  -runTests -testPlatform PlayMode \
  -testResults "$RESULTS/playmode-$STAMP.xml" \
  -logFile "$RESULTS/playmode-$STAMP.log"
```

## Run a specific test filter

```bash
"$UNITY" -batchmode -nographics \
  -projectPath "$PROJECT" \
  -runTests -testPlatform EditMode \
  -testFilter "MyNamespace.MyTestClass" \
  -testResults "$RESULTS/filter-$STAMP.xml" \
  -logFile "$RESULTS/filter-$STAMP.log"
```

## Execute a static method (custom scripts, asset processing)

```bash
"$UNITY" -batchmode -nographics -quit \
  -projectPath "$PROJECT" \
  -executeMethod MyClass.MyMethod \
  -logFile "$RESULTS/exec-$STAMP.log"
```

## Running in the background

Use `run_in_background: true` on the Bash tool call for any test run; compile checks can usually run foreground. **Record the shell id** — never start a second Unity process against the same worktree until that shell has exited (Library lock will fail hard).

## Parsing results

- **NUnit XML** — `<test-run result="Passed|Failed">` at root; failed tests have a `<failure>` child with stack trace
- **Log tail** — `tail -n 100 "$RESULTS/<name>.log"` shows compile errors, stack traces, and the NUnit summary line near the bottom

## Never

- Write logs outside `TestResults/` — stray files in project root or worktree root pollute git status
- Reuse log filenames without a timestamp — older runs get clobbered
- Run two Unity batch processes against the same worktree simultaneously
