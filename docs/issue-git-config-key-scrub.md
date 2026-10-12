# Issue: the credential scrub breaks Git's indexed-config environment protocol

`childEnv()` drops `GIT_CONFIG_KEY_<n>` because the name contains `KEY`, while keeping
`GIT_CONFIG_COUNT` and `GIT_CONFIG_VALUE_<n>`. Git requires that family to be complete —
"any missing key or value is treated as an error" — so **every `git` invocation in a
DSH-spawned shell fails before it starts**.

The scrub also exists in the harness
(`@deepseek-ai/dsh-subprocess/lib/index.js:32`, `scrubbedParentEnv()`), with the same
pattern. This document is written against this repository because this is the layer that
spawns the tool server and the shells, and because it is the copy we can change and ship.

## Impact

Any command executed through `ops_bash` or the bundled PowerShell 7 aborts on the first
git subcommand:

```
error: missing config key GIT_CONFIG_KEY_0
fatal: unable to parse command-line config
```

This is not limited to one repository or one command. `git status`, `git log`, `git add`,
`git commit`, `git push` — all of them, in every DSH session, until the environment is
patched by hand (`GIT_CONFIG_COUNT=0`).

Note the failure is reported as **fatal**, not a warning, and it happens during
command-line configuration parsing, so it precedes any repository work.

## Reproduction

Requires an upstream launcher that injects Git's indexed-config protocol. In our
environment that is Orca (see below), which is also the parent of the harness.

1. Confirm the parent environment carries a complete set:

   ```
   GIT_CONFIG_COUNT   = 2
   GIT_CONFIG_KEY_0   = credential.interactive
   GIT_CONFIG_VALUE_0 = false
   GIT_CONFIG_KEY_1   = credential.guiPrompt
   GIT_CONFIG_VALUE_1 = false
   ```

2. Run any command through a DSH shell tool.

3. Dump `GIT_*` in the child. Observed:

   | Variable | Present in child? |
   | --- | --- |
   | `GIT_CONFIG_COUNT` | yes — `2` |
   | `GIT_CONFIG_VALUE_0` | yes — `false` |
   | `GIT_CONFIG_VALUE_1` | yes — `false` |
   | **`GIT_CONFIG_KEY_0`** | **no** |
   | **`GIT_CONFIG_KEY_1`** | **no** |
   | `GIT_TERMINAL_PROMPT` | yes — `0` |
   | `GIT_ASKPASS` | yes — empty |

   The count and the values survive; only the names are gone.

4. `git config --list` → `fatal: unable to parse command-line config`.

## Root cause

### The scrub is name-based and cannot see protocol structure

`lib/tools.js:49`

```js
/** Credential-shaped environment names the harness never forwards to a child. */
const SENSITIVE_ENV_PATTERN = /KEY|PASSWORD|SECRET|TOKEN/i
```

`lib/tools.js:98`

```js
export function childEnv(parent = process.env) {
  const env = {}
  for (const [key, value] of Object.entries(parent)) {
    if (value === undefined) continue
    if (SENSITIVE_ENV_PATTERN.test(key)) continue      // ← GIT_CONFIG_KEY_0/1 are dropped here
    if (key.toUpperCase().startsWith(DSH_ENV_PREFIX)) continue
    env[key] = value
  }
  return env
}
```

The only caller is `lib/tools.js:425`, inside `FastCtxTools.#connect()`, where the result is
handed to the FastCtx MCP server and, through it, to every shell command.

`GIT_CONFIG_KEY_0` contains the substring `KEY`, so it is treated as a credential.
`GIT_CONFIG_COUNT` and `GIT_CONFIG_VALUE_0` do not, so they are forwarded. The scrub has
no way to know that these three names form one atomic value.

### Git's side of the contract

`git-config(1)`, section ENVIRONMENT
(local copy: `C:\Program Files\Git\mingw64\share\doc\git-doc\git-config.adoc:484`):

```
GIT_CONFIG_COUNT::
GIT_CONFIG_KEY_<n>::
GIT_CONFIG_VALUE_<n>::
	If GIT_CONFIG_COUNT is set to a positive number, all environment pairs
	GIT_CONFIG_KEY_<n> and GIT_CONFIG_VALUE_<n> up to that number will be
	added to the process's runtime configuration. The config pairs are
	zero-indexed. Any missing key or value is treated as an error.
```

So a count with no names is not "config that gets ignored" — it is an error condition by
design. This is the whole reason the failure is fatal rather than cosmetic.

Verified empirically: exporting `GIT_CONFIG_COUNT=1`, `GIT_CONFIG_KEY_0=user.name`,
`GIT_CONFIG_VALUE_0=<any value>` makes `git config --get user.name` return that value.
Git honours these names; they are its own interface, not ours.

### Where the values come from

Upstream of us, **Orca** injects this protocol to disable interactive credential UI:

`C:\Users\admin\AppData\Local\Programs\orca\resources\app.asar.unpacked\out\shared\git-credential-prompt-env.js`

```js
:72  /**
:73   * Disable interactive Git credential UI while preserving cached credentials
:74   * and caller-provided askpass programs.
:75   */
:76  function gitCredentialPromptGuardEnv(env, platform = process.platform) {
:77      const next = appendGitConfigEnv({
:78          ...env,
:79          GIT_TERMINAL_PROMPT: '0',
:80          GIT_ASKPASS: env.GIT_ASKPASS ?? '',
:81          SSH_ASKPASS: env.SSH_ASKPASS ?? '',
:83          GCM_INTERACTIVE: 'never'
:84      },
:87      [
:88          ['credential.interactive', 'false'],
:89          ['credential.guiPrompt',   'false']
:90      ]);
```

The two `false` values match the two orphaned `GIT_CONFIG_VALUE_0`/`_1` observed in the
child exactly, which is how the missing keys were identified. `GIT_TERMINAL_PROMPT=0` and
`GIT_ASKPASS=''` from the same function also survive in the child — they contain none of
the four keywords.

Orca is correct here. Its own validator asserts the family must cross intact:

```js
:8   const GIT_CONFIG_WSLENV_KEY_RE = /^GIT_CONFIG_(?:COUNT|KEY_\d+|VALUE_\d+)$/;
:41  if (!Number.isSafeInteger(count) || indexedKeys.length !== count * 2) { return null; }
:50  const hasDanglingIndex = indexedKeys.some(…)
:93  // config must cross as a complete set or Git rejects the count.
```

## Why this is easy to miss

- **The comment states the assumption that fails.** `lib/tools.js:93-94`:

  > `gh` and `git` keep reading their own configuration files.

  True for configuration *files*. `GIT_CONFIG_*` is configuration carried in the
  **environment**, which is precisely what this function rewrites.

- **The result looks almost right.** Two of three variables survive, so a casual
  `Get-ChildItem env:GIT_*` shows a plausible `GIT_CONFIG_COUNT=2` and two values.

- **The damage is asymmetric.** Only names containing `KEY` are affected, so nothing about
  the *value* of the dropped variable looks sensitive. Nothing here is a credential.

- **Our own git integration never hits it.** `dsh-workspace-changes/lib/types/git.js:40-41`
  forces `GIT_CONFIG_COUNT: '0'` for its own calls and even documents why:

  ```js
  // The subprocess credential scrub removes ambient GIT_CONFIG_KEY_n entries.
  env: { GIT_CONFIG_COUNT: '0', … }
  ```

  So the harness's internal git operations are fine, and the breakage is only visible to
  shells the user or an agent drives.

- **The pattern over-matches in general.** `/KEY/` also catches `GPG_KEY`, `SSH_KEY`,
  `SIGNING_KEY`, and unrelated names such as `KEYBOARD_*`. The git family is the case that
  happens to be load-bearing.

## Suggested fix

Exempt the indexed-config protocol from the name-based scrub. Keep the whole family or
drop the whole family — but do not split it. Keeping is the right choice: none of the
three names is a credential, and dropping them would discard the caller's hardening.

Orca already defines exactly this exemption, so the shape is established:

```js
const GIT_CONFIG_WSLENV_KEY_RE = /^GIT_CONFIG_(?:COUNT|KEY_\d+|VALUE_\d+)$/;
```

Applied here:

```js
const SENSITIVE_ENV_PATTERN = /KEY|PASSWORD|SECRET|TOKEN/i
/**
 * Git's indexed-config protocol (GIT_CONFIG_COUNT plus its zero-indexed
 * GIT_CONFIG_KEY_<n>/GIT_CONFIG_VALUE_<n> pairs) is one atomic value. Its key
 * names contain "KEY" but are config names, not credentials, and Git treats a
 * count with a missing pair as a fatal error. Never split this family.
 */
const GIT_INDEXED_CONFIG_PATTERN = /^GIT_CONFIG_(?:COUNT|KEY_\d+|VALUE_\d+)$/

// in childEnv():
if (SENSITIVE_ENV_PATTERN.test(key) && !GIT_INDEXED_CONFIG_PATTERN.test(key)) continue
```

The same change is needed in `@deepseek-ai/dsh-subprocess/lib/index.js:32` /
`scrubbedParentEnv()`, since that copy does the same filtering for other spawners. Fixing
only one leaves the other path broken.

A narrower `SENSITIVE_ENV_PATTERN` (requiring a word boundary or a recognised suffix
rather than a bare `KEY`) would also reduce the class of false positives, but it is a
separate, riskier change; the exemption above is sufficient and surgical.

## Test gap

`test/mount.test.mjs:262` already covers this function, but its fixture has no git
variables:

```js
await test('FastCtx child drops credentials and DSH facts', () => {
  const env = childEnv({ PATH: 'C:\\bin', HOME: 'C:\\home', GH_CONFIG_DIR: 'C:\\gh', GH_TOKEN: 'fixture', DSH_HOME: 'C:\\dsh', DEEPSEEK_API_KEY: 'fixture' })
  assert.equal(env.PATH, 'C:\\bin'); assert.equal(env.GH_CONFIG_DIR, 'C:\\gh')
  for (const key of ['GH_TOKEN', 'DSH_HOME', 'DEEPSEEK_API_KEY']) assert.equal(Object.hasOwn(env, key), false)
})
```

Extending the fixture with a complete protocol would have caught it:

```js
const env = childEnv({
  PATH: 'C:\\bin',
  GIT_CONFIG_COUNT: '2',
  GIT_CONFIG_KEY_0: 'credential.interactive',
  GIT_CONFIG_VALUE_0: 'false',
  GIT_CONFIG_KEY_1: 'credential.guiPrompt',
  GIT_CONFIG_VALUE_1: 'false',
})
for (const key of [
  'GIT_CONFIG_COUNT',
  'GIT_CONFIG_KEY_0', 'GIT_CONFIG_VALUE_0',
  'GIT_CONFIG_KEY_1', 'GIT_CONFIG_VALUE_1',
]) assert.equal(env[key] !== undefined, true, `${key} must survive intact`)
```

A companion case should assert the family is never split: if the protocol is preserved at
all, `COUNT` and every pair up to it must be present.

## Evidence summary

| Fact | Where |
| --- | --- |
| Scrub pattern | `lib/tools.js:49` |
| Scrub implementation and its sole caller | `lib/tools.js:98`, `lib/tools.js:425` |
| The assumption that fails | `lib/tools.js:93-94` |
| Git's "missing key is an error" rule | `git-config(1)`; local `git-config.adoc:484-498` |
| Protocol injected by the launcher | Orca `out/shared/git-credential-prompt-env.js:76-100` |
| Launch side treats the family as atomic | same file, `:8`, `:41`, `:50`, `:93` |
| Harness copy of the same pattern | `@deepseek-ai/dsh-subprocess/lib/index.js:32`, `:50-52` |
| Harness workaround that hides it internally | `dsh-workspace-changes/lib/types/git.js:40-41` |
| Existing test that misses this | `test/mount.test.mjs:262-266` |

## Workaround

Until this ships, set `GIT_CONFIG_COUNT=0` in the child. Git then ignores the orphaned
value pairs and behaves normally.

```bash
export GIT_CONFIG_COUNT=0     # bash / ops_bash
```

```powershell
$env:GIT_CONFIG_COUNT = '0'   # bundled PowerShell 7
```

A PowerShell profile fix does not help: the shells are spawned with `-NoProfile`, so no
profile is loaded.
