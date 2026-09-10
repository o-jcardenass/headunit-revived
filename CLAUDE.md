# CLAUDE.md: ohu-transfer (testing agent orientation)

This file orients the **testing agent** that works in this directory: a git worktree holding the
transfer branch, an orphan hand-off channel between a planning agent on a PC and a physical test
rig with real Android head units and phones on adb. It does not repeat how to run a round.
**`TESTING-TEMPLATE.md` is the operational authority for that** (capture rules, `settings.xml`,
the automation surface, the clean-run protocol, install discipline, verdicts, rig quirks). Read it
before touching a device, and re-read it if a round feels unfamiliar; this file only covers what it
does not: how to change app code when a round needs that, and how to commit and report back on this
branch.

## Layout

Two worktrees of one repository, sharing an object store:

- `/home/ocardenas/projects/ohu-project/code-review-ohu/open-headunit`: the app, `main` and
  feature branches. All code changes happen here.
- `/home/ocardenas/projects/ohu-project/code-review-ohu/ohu-transfer`: this worktree, the
  `transfer/hotspot-unreadable-config-results-20260807` branch. Nothing here is ever merged into
  the app; it is a message channel, not a codebase.

Remotes on the shared object store: `origin` is `andreknieriem/open-headunit` (upstream, read-only
in practice), `fork` is `o-jcardenass/open-headunit` (where our branches, including this transfer
branch, are pushed).

Start every round the way the template says: read `README.md`'s Threads table to find your thread
and the brief it names, read only that brief (not other threads', per the README), and inventory
`hur-wifi-test-scripts/` on the rig before building anything.

## Modifying app code

A round sometimes needs a code change: a log line the brief asks for, a build break to fix, or a
small correction a round's own finding proves. Do the edit in the app worktree, never in this one
(there is no app code here to edit). The conventions below are condensed from the app repo's own
`CLAUDE.md`; read that file directly if a rule here feels incomplete, but do not restate it back
here.

- **Build:** `./gradlew :app:assembleGithubDebug` is the only APK build that works. **Test:**
  `./gradlew :app:testGithubDebugUnitTest`. Two product flavors on the `distribution` dimension,
  `github` and `playstore`; only `github` is buildable here. Do not add flavor-specific code for a
  rig round.
- `applicationId` `com.andrerinas.headunitrevived` and namespace/package
  `com.andrerinas.openheadunit` differ on purpose (the app kept its Play Store identity across a
  rename). Never "fix" that mismatch.
- **Logging is `AppLog`, never `android.util.Log`**, prefixed with the class name. Users attach
  these logs to bug reports, so a line added for a round should stay informative after the round is
  over. A log call's priority is not its level: a line can be `AppLog.d` and still sit behind an
  `if (AppLog.LOG_VERBOSE)` guard, so grep the guard, not the call, before writing a brief around
  what a log level should surface.
- **Pure decidable logic goes in a Kotlin `object` with a JUnit4 test in the same package**, under
  `app/src/test/`. Robolectric is not used, so anything touching an `Activity` or a `View` cannot be
  unit tested and has to be verified on the rig instead; that is exactly the coverage a round exists
  to provide.
- **Comments are 1-2 lines, 4 is the hard ceiling.** State the hazard and what the code does about
  it, never history or measurements. `[FIX]` / `[BUG_FIX]` markers stay; read the comment before
  touching code near one, several encode hardware workarounds a round already paid for once.
- **Never touch `app/src/main/proto/` during a round.** Editing a `.proto` alone breaks the build:
  the generated Java is committed and hand-regenerated with a pinned `protoc`. That is planning-side
  work, not something a rig round does.
- `CHANGELOG.md` and `README.md` in the app repo belong to the upstream maintainer. Never edit
  them. The app repo's own `CLAUDE.md` is local-only and never goes into a PR.
- **Commits are grouped by component, one thing per branch, no microcommits.** If a round's fix
  touches two unrelated areas, that is two commits, not one.

**Scope discipline:** if what a round needs is more than a log line or a build fix, say so in
Setup notes and in the results file instead of quietly widening the change. The planning agent
owns the branch's shape; a testing agent that grows a fix on its own has changed what the next round
is measuring without anyone deciding that on purpose.

## Committing on this branch

A `PreToolUse` hook, `open-headunit/.claude/hooks/commit-guard.py`, checks every `git commit`
before it runs. It is scoped to this repository's object store, so it applies in **both**
worktrees, including this one. It blocks a commit whose message:

- is more than 5 lines (subject, blank line, one or two body lines);
- has a subject over 80 characters, or a subject not shaped `Scope: what changed` (for example
  `WiFi Direct: ...`, `Native AA: ...`);
- has a non-blank second line;
- contains an em dash;
- carries a `Co-Authored-By: ... Claude` or `Claude-Session:` trailer, or a "Generated with Claude
  Code" footer;
- **on a branch whose name starts with `transfer/` (this one), contains `#<digits>`**: house rule 7,
  below.

A blocked commit prints its reasons on stderr, so treat that output as the fix list rather than
retrying blind.

Real examples from this branch's own log:

```
native-aa-recovery-identity-and-speed: round 3 results - R1 FAIL both HUs (3 non-phone
AG-record devices classified PHONE), standing selector crash, round stopped

ultrawide-touch-alignment: round 8 candidate SHAs corrected
```

The subject carries the topic slug, the round, and what the file is (`brief`, `results`,
`addendum`), optionally followed by ` - <short outcome>`. Keep the body to the one or two lines the
guard allows; anything longer belongs in the results file itself, not the commit message.

**House rule 7, no issue or discussion numbers anywhere on this branch.** Describe a thread by name
and cite a SHA, a log line or an evidence filename instead of a tracker number: this channel has no
access to the tracker, and a number dates badly next to the file it is naming. The guard enforces
this mechanically for commit messages; hold file contents to the same rule by habit. The one file
naming pattern that looks like an exception, `pr<NNN>-review-*.md`, names a pull request under
review rather than pointing at a tracker issue, and is not a case for citing an issue number
elsewhere.

**Append-only once pushed.** A brief or a results file that has been pushed to `fork` is never
force-pushed and never rewritten. A correction is a new commit on top, as in the round 8 example
above. Proofread a file before its first push; after that, fixing it is another commit, not an
edit to history.

## Reporting back

One results file per round, `<topic>-round<N>-results.md`, committed next to its brief. Follow the
skeleton in TESTING-TEMPLATE.md §7 exactly rather than improvising a shape: the header block
(Candidate, Baseline, APK md5s, Unit, Date), one `## Setup notes` section, one `## R<id>` section
per run with a bolded verdict, and a closing `## Anything the brief did not ask about`. Do not
duplicate that skeleton here; if a results file looks wrong, compare it against §7, not against this
paragraph.

**Setup notes is not a formality.** Every deviation from the brief, every wrong settings key,
every log string that does not match, every step that could not be performed as written goes there.
Most of what TESTING-TEMPLATE.md itself now knows about this rig came out of a previous round's
Setup notes; a round that skips it costs the next round the same hours.

Verdicts are exactly one of **PASS**, **FAIL**, **INCONCLUSIVE**, **UNTESTABLE**, per §6. For a
FAIL, attach the full capture, never an excerpt. Give the measurement, never an adjective:
"5180 MHz", not "5 GHz".

Evidence goes in `evidence/<topic>-round<N>/`. A photo-heavy round also gets a sibling
`<topic>-round<N>-photos/` directory next to the results file, as several `ultrawide-touch-alignment`
rounds do.

**The README is a router, not a log.** It carries one row per thread in the `## Threads` table,
`| Thread | State | Next |`. A round's outcome belongs in its own results file, never spelled out
as a narrative in the README. When a round finishes, update that thread's row and name the results
filename in it; that is the entire edit the README gets.
