# Fork Deltas — divergence from `pingdotgg/t3code`

This fork (`chidi21/t3code`) is intentionally diverging from upstream. This file is a running log of every patch the fork carries **on top of** upstream that must be **preserved or re-applied after every upstream merge**.

## How to use this during an upstream merge

1. Before merging, read this file top-to-bottom.
2. Merge `upstream/main` into `main` as usual.
3. For each entry below whose **Status** is `active`, check whether upstream re-introduced the problem (look at the **Upstream trigger commit** and read the file(s) listed under **Touches**).
4. If upstream still has the bug, re-apply the fix as described in **Re-apply recipe**.
5. If upstream has fixed it a different way, mark the entry `superseded by <upstream sha>` and remove the local patch so we don't carry a redundant delta.
6. Run the verification commands in each entry.

## Entry template

```
### <short title>

- Fork commit:     <sha>  (<branch>)
- Upstream trigger: <sha>  (<PR #, message>)
- Introduced in:   <upstream merge that pulled the regression in — e.g. 991c753f>
- Status:          active | superseded by <sha> | reverted
- Platforms:       windows | macos | linux | all
- Touches:         <file:line>, ...
- Problem:         <one-paragraph description of the symptom and root cause>
- Fix:             <one-paragraph description of our change>
- Re-apply recipe: <exact edit instructions if a future merge undoes this>
- Verify:          <commands that must pass on each affected platform>
```

---

## Active entries

### Windows: `apps/server/scripts/cli.ts` must spawn `process.execPath` without `shell: true`

- **Fork commit:** `f4bd9947` (`main`)
- **Upstream trigger commit:** `8dba2d64` "Adopt Node-native TypeScript for desktop and server (#2098)" — author: Julius Marminge, merged into upstream `main` 2026-04-16
- **Introduced in:** upstream merge `991c753f` (`Merge branch 'pingdotgg:main' into main`, 2026-04-22)
- **Status:** active
- **Platforms:** windows (the change is a no-op on macOS/Linux, where `shell` was already `false` because `process.platform !== "win32"`)
- **Touches:** `apps/server/scripts/cli.ts:150` (the `buildCmd` spawn only — **do not** change the `publishCmd` spawn of `npm` at ~L246)

**Problem.** Commit `8dba2d64` changed the tsdown build step from

```ts
ChildProcess.make({ ..., shell: process.platform === "win32" })`bun tsdown`
```

to

```ts
ChildProcess.make(process.execPath, ["--run", "build:bundle"], {
  ..., shell: process.platform === "win32",
})
```

On Windows, `process.execPath` resolves to `C:\Program Files\nodejs\node.exe`. With `shell: true`, Node concatenates the command and args into a single string and hands it to `cmd.exe`, which splits on the space in `Program Files` and reports:

```
'C:\Program' is not recognized as an internal or external command
```

`bun run build` exits non-zero at `t3#build`, so `apps/server/dist/bin.mjs` and `apps/server/dist/client/index.html` are never produced, which in turn blocks `bun run dist:desktop:win`.

**Fix.** Disable `shell` for this spawn. `process.execPath` is an absolute `.exe` path, so it never needs PATH/shim resolution:

```ts
// apps/server/scripts/cli.ts, buildCmd
ChildProcess.make(process.execPath, ["--run", "build:bundle"], {
  cwd: serverDir,
  stdout: config.verbose ? "inherit" : "ignore",
  stderr: "inherit",
  // NOTE: do not enable `shell` here. process.execPath is an absolute exe
  // path (e.g. "C:\Program Files\nodejs\node.exe") — routing it through
  // cmd.exe with shell: true splits the unquoted path on the space and
  // breaks the build on Windows. No PATH/shim resolution is needed.
}),
```

The `publishCmd` spawn (line ~246) legitimately runs `npm`, which _is_ a `.cmd` shim on Windows, so `shell: true` there is correct — **leave it alone**.

**Re-apply recipe.** If upstream re-emits the pre-fix code after a future sync:

1. Open `apps/server/scripts/cli.ts`.
2. Locate the `buildCmd` block (around the `[cli] Running tsdown...` log line).
3. Delete the line `shell: process.platform === "win32",` inside the `ChildProcess.make(process.execPath, ...)` call.
4. Replace the `// Windows needs shell mode...` comment with the four-line NOTE above.
5. Leave the `publishCmd` spawn unchanged.

**Verify.**

```bash
bun install
bun fmt:check      # must pass
bun lint           # 0 errors
bun typecheck      # 10/10 packages
bun run build      # must pass on Windows; produces apps/server/dist/bin.mjs + dist/client/index.html
```

Also smoke test on macOS with `bun run build` to confirm no regression on Unix — `shell` was already `false` there, so behavior must be unchanged.

---

## Superseded / reverted entries

_(none yet)_

## Known-broken-upstream, not yet patched

Things we have identified as Windows-unfriendly in upstream but have **not** fixed in this fork. Capturing so they are not forgotten, and so future merges don't mistake them for regressions:

- **53 test failures in `t3#test`** on Windows as of merge `991c753f` (2026-04-22). Breakdown:
  - 22 failures in entirely new Cursor/ACP test files added by upstream `9c64f12e Add ACP support with Cursor provider (#1355)` — `CursorAdapter.test.ts` (15), `CursorProvider.test.ts` (3), `CursorTextGeneration.test.ts` (4). These have never run on Windows — not a regression.
  - 15 failures in `CodexTextGeneration.test.ts` — mock `codex` binary is written without a `.exe` extension, so `cmd.exe` can't execute it (`'...\bin\codex' is not recognized`). Pre-existing file, platform-specific test fixture issue.
  - 4 failures in `ClaudeTextGeneration.test.ts` — `Invalid JSON provided to --settings`, likely Windows CLI arg-quoting of JSON payloads.
  - 2 failures in `bootstrap.test.ts` — `EBADF` on fd close. `resolveFdPath(fd, "win32")` returns `undefined` by design, so the code falls back to inherited-fd reads that race with `closeSync`. Pre-existing Windows fd-semantics limitation.
  - 7 scattered failures across `server.test.ts`, `config.test.ts`, `keybindings.test.ts`, `CheckpointReactor.test.ts`, `ServerSecretStore.test.ts`, `ProjectFaviconResolver.test.ts`, `integration.test.ts`.
- None of the above were caused by the upstream sync reverting fork work — fork commit `e2b34efb` only added docs/config. If we decide to fix any of these, add a new **Active entry** above when the fix lands.
