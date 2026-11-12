# Build your own foundry

A foundry is a build line for Claude Code that runs while you sleep. You write tickets. A script on a timer picks one, gives Claude a clean copy of the code to work in, checks the result with plain scripts, has a fresh Claude session review it, then merges it. In the morning you read reports, not diffs.

This is how ours works, written so you can build your own. It is a guide, not a product: no install, no framework, a few hundred lines of shell when you are done. There is a working sketch of the dispatcher at the end.

It goes with Building with Claude #01.

---

## The line

Twelve steps. A person does two of them, Claude does three, scripts do the rest.

| # | step | who |
|---|---|---|
| 1 | write the spec | a person |
| 2 | write the ticket, with a check that fails today | Claude (from the spec) |
| 3 | lint the ticket at filing | script |
| 4 | queue it | script |
| 5 | cut a fresh worktree from the remote | script |
| 6 | build | Claude |
| 7 | gate: report, check lines, old tests | script |
| 8 | run the full test suite | script |
| 9 | review, in a fresh session | Claude |
| 10 | merge the base in, test again, push one commit | script |
| 11 | deploy staging | script |
| 12 | ship to production | a person says "ship", a script builds it from the remote |

The idea underneath all of it: **the model is the expensive, unreliable step, so every cheap, certain check runs before it and after it.** Scripts decide pass or fail wherever a script can. Claude does the parts that need judgement.

---

## What you need

- A git repo with a remote (GitHub or a bare repo on another disk). The remote is the truth; nothing ships from a local checkout.
- One test command that runs the whole suite and exits non-zero on a failure. If you do not have one, build that first. A foundry without tests merges whatever it is given.
- Claude Code, run headless with `claude -p`.
- A machine that stays on, and a timer (cron, launchd or systemd).
- A machine or user account with nothing on it you cannot lose. The build runs unattended, so it cannot stop to ask your permission for each command. Keep production keys out of its environment.

---

## 1. Spec first

Before anything is ticketed, write down what the thing is for. In this order:

0. **One line.** "X lets WHO do WHAT so that OUTCOME." If it needs "and", it may be two features.
1. **Needs.** What hurts today, with evidence: a number, a date, a real example.
2. **Jobs.** "When I am in situation, I want to do something, so I can get result." Three to seven of them.
3. **Features.** Each one points at a job, and is marked v1, later or never. "Never" stops the build drifting.
4. **Rules.** What must it never do? What needs a person's yes?

Then, and only then:

5. **Architecture.** One line of flow, `trigger > input > part > part > storage > output > who sees it`, and a table of the parts: what each does, where it runs, whether it reuses something you have.
6. **Failure and proof.** For each part: how it breaks, who notices, and the command that proves it works.
7. **Build plan.** Phases, each ending in something you can use. This is where the tickets come from.

Give the spec a status line. Nothing gets ticketed until it says ready. A change of mind after that goes into the spec first, dated, then into tickets.

**Why:** needs decide features and features decide architecture. Start at the architecture and you build the wrong thing very efficiently. A model builds exactly what the ticket says; the spec is where you make sure the ticket says the right thing.

Honest note: in ours this is a habit and a template, not code. Nothing refuses a ticket that has no spec.

---

## 2. The ticket

One markdown file per ticket. Frontmatter is the state; the body is the job. The part that matters most is the check block.

````markdown
---
id: T-042
title: retry the export when the API times out
status: ready
size: m
---

## Job
When the export call times out, retry twice with a backoff, then fail with a clear error.
Today one timeout kills the whole nightly export.

## Files
- src/export.py
- tests/test_export.py

## Done when
A timeout on the first call still produces a full export; three timeouts produce
the error "export failed after 3 attempts" and exit code 2.

```check
python -m pytest tests/test_export.py -k "timeout" -q
grep -q "export failed after 3 attempts" src/export.py
```
````

Rules for the check block:

- Every line is a shell command that exits 0 on success.
- Each line runs on its own, from the repo root, with no input. Do not chain them into one script: `set -e` ignores a failure on the left of `&&`, and a `cd` in one line leaks into the next.
- Include at least one check for the thing that must **not** happen (the refused input, the skipped row, the blocked send). Happy-path-only tests are how a feature passes and still breaks.

---

## 3. Lint at filing: a check that fails today

When a ticket is filed, a script refuses it if:

- there is no check block, or the repo has no test command, or
- **every check line already passes on today's code.**

The second one is the important one. Run each check line in a throwaway worktree of the remote base. If they all pass before anyone has built anything, the block proves nothing: the build could do nothing and still go green. One line must fail today and pass after.

```bash
# red at base: refuse a ticket whose checks already pass
wt=$(mktemp -d)
git -C "$REPO" worktree add -q --detach "$wt" "$REMOTE/$BASE"
fails=0
while IFS= read -r line; do
  ( cd "$wt" && bash -o pipefail -c "$line" ) </dev/null >/dev/null 2>&1 || fails=$((fails+1))
done < <(checks "$TICKET")
git -C "$REPO" worktree remove --force "$wt"
[ "$fails" -gt 0 ] || { echo "refused: every check already passes, so it proves nothing"; exit 1; }
```

Allow an explicit, written exception for the rare check that should stay green (a count that must hold, a release step).

Also refuse a check line that cannot run at all (a missing tool, a path that does not exist). Fix the check, not the code.

---

## 4. The queue

Keep it boring: a `QUEUE.md` with ticket ids in priority order, and the status in each ticket's frontmatter (`ready`, `building`, `review`, `done`, `blocked`, `retry`). The dispatcher takes the first ready ticket in queue order.

Have one writer for the queue. Two sessions reordering the queue at once is how a ticket gets built twice.

---

## 5. A worktree per ticket, cut from the remote

```bash
git -C "$REPO" fetch -q "$REMOTE" "$BASE"
git -C "$REPO" worktree add -q -b "foundry/$ID" "$WORK/$ID" "$REMOTE/$BASE"
```

- Fresh from the **remote** base, never from your local checkout. Your local copy has half-finished edits in it.
- Its own folder and branch, so several tickets can build at once without touching each other or you.
- Claude commits inside it. Claude never pushes. Only the dispatcher merges and pushes.
- Remove the worktree and its branch when the run ends, on every exit path: done, failed or blocked. The report and logs keep what you need; the code is on the remote or in the next attempt's fresh copy. Skip this and you have hundreds of stale folders within a month. Run `git worktree prune` now and then for any a crash left behind.

```bash
git -C "$REPO" worktree remove --force "$WORK/$ID"
git -C "$REPO" branch -D "foundry/$ID"
```

Ours tells the build session "never push" in its prompt. That is an instruction, not a lock. The stronger version is a `pre-push` hook on the shared branch that runs the test suite, so a push from anywhere else has to pass too. Do as I say here, not as I did: install the hook.

---

## 6. The build session

```bash
( cd "$WT" && timeout 5400 claude -p "$PROMPT" --permission-mode bypassPermissions ) > "logs/$ID.build.log" 2>&1
```

The prompt carries: the ticket, your standing rules (one file, kept short, pasted into every session), where it is and what it may not touch, the time cap, the test command, and the report format.

The report is the product as much as the code is. Ask for:

- **Summary** a non-programmer can read.
- **Status table**, each item marked one of: **written** (the code exists, unproven), **proven** (it ran: failing before, passing after, with the numbers), **live** (running in the real system). Never blur them. "It parses" is not proof.
- **What I ran**, with the key output lines, so the reviewer can re-run them.
- **Open issues**: anything unproven or skipped.
- **Oracle changes**: one line per existing test file it modified or deleted, `path: why`.

Kill it at the time cap. When the session runs shell commands, killing the parent is not enough: kill the whole process tree, or a stuck child outlives the cap and the slot hangs. Also stop a build whose log has stopped growing for a long time; something is waiting for an answer nobody will give. Set the limit above your slowest quiet step: ours started at 20 minutes and went to 45 after a 32-minute test suite that prints nothing tripped it on green builds.

---

## 7. The gate: scripts before the model

After the build and before any review, a plain script checks:

1. **The ticket was not edited.** Copy the ticket and its check block before the build; compare after. If the builder rewrote its own exam, undo it and block the ticket.
2. **There is a report**, in the right shape.
3. **Every check line passes**, each run on its own in the worktree.
4. **Old tests were not quietly changed.** `git diff --name-only --diff-filter=MD` for test files; each one must be named under Oracle changes with a reason, or the build goes back. Git decides this, not a model.

```bash
changed=$(git -C "$WT" diff --name-only --diff-filter=MD "$BASE_SHA" HEAD \
  | grep -E '(^|/)tests?/|_test\.|\.test\.' || true)
for f in $changed; do
  grep -qF "$f:" "$REPORT" || fail "test file changed with no reason in the report: $f"
done
```

Then run the full test suite. Red here goes back to the build.

**Why this order:** a failed review costs a rebuild, a suite run and a second review. A failed shell check costs seconds. Ask the script first and the model last.

---

## 8. The review: a fresh session, never the author

A new `claude -p`, no resume, its own prompt, run in the same worktree. Tell it to be adversarial. Its job is the gap between what the report claims and what is true.

In order:

1. Run the check block itself. A failing check is an automatic fail.
2. Read the diff (`git diff $BASE_SHA...HEAD`): secrets, `.env` files, deleted or weakened tests, sends or deploys the ticket did not ask for, hard-coded paths.
3. For every row the report marks **proven**, re-run the cheapest command that would expose a false claim. A proof it cannot reproduce is a fail. A row honestly marked "written" is fine.
4. Every rule in the ticket has a negative test.
5. Scope: did it do the ticket, or something next to it? Narrowing the scope without saying so is a fail. Saying so with a reason is fine.
6. Code quality last, and only as notes: a much simpler shape, a helper that duplicates one you already have. These never fail a build on their own, except a new dependency nobody asked for, or a stub left in the code and not listed.

Force the answer into a fixed shape so a script can read it:

```bash
SCHEMA='{"type":"object","properties":{"verdict":{"type":"string","enum":["pass","fail"]},"reasons":{"type":"array","items":{"type":"string"}}},"required":["verdict","reasons"]}'
( cd "$WT" && timeout 1800 claude -p "$REVIEW_PROMPT" --output-format json --json-schema "$SCHEMA" ) > "logs/$ID.review.json"
VERDICT=$(jq -r '.structured_output.verdict // (.result | try fromjson | .verdict) // empty' "logs/$ID.review.json")
```

A fail sends its reasons back for one rebuild. Fail twice and the ticket is blocked for a person.

"Fresh session" means a new session of the same model, with no memory of writing the code. We do not use a second, different model as the reviewer. It sounds stronger than it is. We do send the riskiest tickets (logins, secrets, anything that sends) to a stronger model for review.

---

## 9. Merge, test again, push one commit

```bash
git -C "$WT" fetch -q "$REMOTE" "$BASE"
git -C "$WT" merge -q --no-edit "$REMOTE/$BASE"        # what landed while it built
( cd "$WT" && bash -o pipefail -c "$TEST_CMD" )       # green alone is not green together
git -C "$WT" reset -q --soft "$REMOTE/$BASE"          # squash: one commit per ticket
git -C "$WT" commit -q -m "$ID: $TITLE"
git -C "$WT" push -q "$REMOTE" "HEAD:$BASE"           # refused if the base moved: retry next run
```

Two tickets can each pass alone and break together. Merging the base in and testing again catches it before the push, not after. One commit per ticket means one revert undoes one ticket.

---

## 10. Staging by script, production by a person

- After the push, a script deploys staging.
- Production waits for a person to say "ship" for that ticket. Record the yes somewhere a script can read it (a line in a decisions file), and have the ship step refuse to start without it.
- The ship script fetches the remote and deploys exactly that ref (`git archive` of the remote ref is enough). Never deploy from a checkout someone might have touched.

---

## 11. Do not deploy code you have not tested as that exact code

A full green test run writes a stamp named after the code's git tree id:

```bash
TREE=$(git rev-parse HEAD^{tree})
git diff --quiet && git diff --cached --quiet || { echo "uncommitted changes: no stamp"; exit 1; }
"$TEST_CMD" && touch ".tested/$TREE"
```

The deploy script looks for the stamp for the tree it is about to deploy, and refuses without it. A failed run, uncommitted changes, or a missing test database: no stamp, no deploy. Allow an override only with a written reason, and log it.

In ours this guards one deploy path so far, not all of them.

---

## Rules we learned the hard way

- **DO write the spec first.** Needs, jobs, features, rules; architecture after.
- **DO give every ticket a check that fails today.** Refuse it at filing otherwise.
- **DON'T let the builder edit the tests quietly.** Snapshot the ticket; name every changed test file with a reason; let git decide.
- **DO ask the script first and the model last.** Every cheap, certain check before every expensive, uncertain one.
- **DON'T let the author review its own build.** A fresh session, a fixed answer format, re-run every claimed proof.
- **DON'T deploy code you have not tested as that exact code.** A stamp per tree id.
- **DON'T add a model on faith. Count what merges.** We tried two cheaper models for builds. One built none of the 257 tickets that merged over two weeks; the other had 9 of its 12 builds sent back. Both are off. A cheaper build that bounces costs more than a dear one that lands.
- **Report written, proven, live.** Never let "I wrote it" pass for "it works".
- **Every no becomes a rule.** When you reject something the foundry made, write the reason as one dated line in the standing rules file, so no later ticket repeats it. Keep that file short; it is pasted into every session.
- **Read the morning digest, not the code.** If you need to read a diff to trust a merge, the gate is missing a check.

---

## Once the basics run

The line above is the core. These are the pieces we added after it ran, each because something went wrong without it. Add them in roughly this order.

### Make the build map before it edits

The slop that hurts is code that works in the file it changed and breaks a caller nobody looked at. So before its first edit, the build writes a map into its report:

- **Anchors:** the files and functions the ticket lands on.
- **Bridges:** every seam the change crosses, one line each: `A -> B (call, import, table, route, env, job)`. Callers, callees, database tables, API routes and their clients, queues and timers, config.
- **Blast radius:** what could break if an anchor changes, and the test covering each bridge, or the words "no test".

The review then checks the diff against the map. A changed file the map does not explain is a fail. A bridge changed on one side with nothing proving the other side is a fail.

### No half contracts

When a report says "the next ticket will build the other half", it must name that ticket, and that ticket must own a test that runs both halves. A script reads the report for those phrases and sends the build back without the name. Two halves that each pass their own tests and were never run together are the most common way a "done" feature turns out not to work.

### Show the reviewer the neighbours

Give the review the open issues of the last few tickets that touched the same files. If a neighbour said "the reader of this file comes later, keep the format", the review checks this diff kept it.

### Turn lessons into review steps

Keep a short file of lessons, each with an "applies to" tag (a migration, an API route, a scheduled job, a paid API). Paste the matching ones into the review prompt, and make the reviewer cite a `file:line` showing each lesson is followed, or show it does not arise. A lesson nobody checks is a lesson you learn twice.

### Build, don't run

The build may write a scheduled job, a bot or a sender; it may not start one. A live start (loading a timer, sending, running a bot) fails the review unless the ticket says live and a person has written the yes for that ticket, in the same decisions file as "ship".

### A runaway stop

Watch the build's transcript as it runs (`claude -p --output-format stream-json` writes one line per event). Stop the build after a run of failed tool calls in a row (ours: 5) or too many requests in one session (ours: 250). Then commit what it did as work in progress, write the report from the transcript with a script, and block the ticket. Do not spend a model on writing up a runaway.

### Pace against your usage limits

An unattended line will happily spend your whole week by Tuesday. Give it a budget and keep it small: **start at 25% of your usage, no more.** Before each build, read how much of your usage is spent (the stream-json transcript includes rate limit events; keep the latest in a file). Only start a build when the foundry's share of the week is under a straight line to 25% at the reset, and its share of the current 5-hour window is under 25% too. At 25% it stops until the next reset, whatever is in the queue. If the reading is stale or missing, do not build: an unknown meter means no. A session that hits the limit puts its ticket back as ready, with its attempts unchanged. The rest of your usage is yours for working by hand; raise the cap only once you have watched a few weeks of what it actually spends.

### A priority lane

One flag on a ticket lets it, and every ticket it waits on, jump to the front of the queue. It still waits for the usage budget, and every other gate still applies. Use it rarely.

### Run builds side by side, except on the line itself

Several tickets can build at once in their own worktrees (ours: 2 per repo). A ticket that changes the dispatcher's own files runs alone: a build that edits the script that is running it is a bad afternoon. Each ticket lists the paths it touches, so the check is a script, not a guess.

### Test what changed per merge, everything per night

After a merge, run only the suites the change touches: each suite has a `# covers:` line listing the files it covers, plus a fast core that always runs. Run the full suite once a night and mail the names of anything red. A merge that takes half an hour to test is a queue that never empties.

### The riskiest tickets get the strongest reviewer

Tickets that touch logins, secrets, money or anything that sends are picked out by a word list and reviewed by a stronger model, capped at a few a day. Everything else gets a fresh session of the build model.

### Keep the sessions lean

Every turn resends the whole session so far, so every file read is paid for again on every later turn. Tell the build: search first, then read only the lines it needs; batch independent commands; trim long output; budget tool rounds by the ticket's size. Give small tickets a slimmer prompt.

### A morning digest

One message a day: what merged, what is waiting on you and for what, what broke, and how much of the week's usage is spent. You answer the things waiting on you with one word each. If you find yourself reading logs instead, the digest is missing a line.

---

## Appendix: a dispatcher sketch

One run takes one ready ticket through the line. Call it from a timer every 30 minutes. It is a sketch to read and adapt, around 100 lines: no usage pacing, no stall or runaway watcher, no staging deploy, none of the extras above. It was tested end to end against a stand-in for `claude` (pass, gate fail and review fail paths), not against a real repo of yours.

Needs bash, git, jq and `timeout` (on macOS: `brew install coreutils`).

Layout it expects:

```
foundry/
  foundry.sh
  QUEUE.md          ticket ids, one per line, in priority order
  tickets/T-001.md    one file per ticket (format in section 2)
  prompts/build.md    your build prompt
  prompts/review.md   your review prompt
  reports/ logs/ state/ work/
```

```bash
#!/usr/bin/env bash
# foundry.sh: take ONE ready ticket through the line. Run it from a timer.
set -uo pipefail
: "${REPO:?path to your main checkout}" "${TEST_CMD:?one command that runs the whole suite}"
REMOTE=${REMOTE:-origin}; BASE=${BASE:-main}; CLAUDE=${CLAUDE:-claude}
BUILD_SECS=${BUILD_SECS:-5400}; REVIEW_SECS=${REVIEW_SECS:-1800}; MAX_ATTEMPTS=${MAX_ATTEMPTS:-2}
cd "$(dirname "$0")"; F=$PWD; mkdir -p reports logs state work

mkdir state/lock 2>/dev/null || exit 0          # one dispatcher at a time
trap 'rmdir "$F/state/lock"' EXIT

get()     { sed -n "s/^$2: *//p" "$1" | head -1; }
set_st()  { sed -i.bak "s/^status: .*/status: $2/" "tickets/$1.md" && rm -f "tickets/$1.md.bak"; }
checks()  { awk '/^```check/{f=1;next} /^```/{f=0} f && NF && !/^#/' "$1"; }
log()     { echo "$(date '+%F %T') $*" >> logs/foundry.log; }
fill()    { # fill PROMPT_FILE: the prompt with its {{placeholders}} filled, then the ticket
  local t; t=$(cat "$1")
  t=${t//'{{ID}}'/$ID}; t=${t//'{{WT}}'/$WT}; t=${t//'{{REPORT}}'/$REPORT}
  t=${t//'{{TEST}}'/$TEST_CMD}; t=${t//'{{BASE_SHA}}'/$BASE_SHA}
  printf '%s\n\n%s\n' "$t" "$(cat "state/$ID.ticket")"; }
cleanup() { git -C "$REPO" worktree remove --force "$WT" >/dev/null 2>&1; git -C "$REPO" branch -D "foundry/$ID" >/dev/null 2>&1; }
fail()    { # fail REASON [retry]: back to the queue while attempts last, else blocked for a person
  local st=blocked; [ "${2:-}" = retry ] && [ "$ATT" -lt "$MAX_ATTEMPTS" ] && st=retry
  cp "state/$ID.ticket" "tickets/$ID.md"; set_st "$ID" "$st"
  printf '%s\n' "$1" > "state/$ID.reasons"         # the next attempt is told why
  printf '\n## Foundry: %s (%s)\n%s\n' "$st" "$(date '+%F %T')" "$1" >> "$REPORT"
  log "$ID $st: $1"; cleanup; exit 0; }

# pick: first ticket in queue order that is ready or retry
ID=""
for id in $(grep -oE '^[A-Z]+-[0-9]+' QUEUE.md); do
  st=$(get "tickets/$id.md" status 2>/dev/null)
  [ "$st" = ready ] || [ "$st" = retry ] && { ID=$id; break; }
done
[ -n "$ID" ] || exit 0
ATT=$(get "tickets/$ID.md" attempts); ATT=$(( ${ATT:-0} + 1 ))
grep -q '^attempts:' "tickets/$ID.md" || sed -i.bak "s/^status: .*/&\\
attempts: 0/" "tickets/$ID.md"
sed -i.bak "s/^attempts: .*/attempts: $ATT/" "tickets/$ID.md"; rm -f "tickets/$ID.md.bak"
set_st "$ID" building; cp "tickets/$ID.md" "state/$ID.ticket"   # snapshot: the exam is frozen now
TITLE=$(get "state/$ID.ticket" title); REPORT=$F/reports/$ID.md; WT=$F/work/$ID; BASE_SHA=""
log "$ID start (attempt $ATT): $TITLE"
[ -n "$(checks "state/$ID.ticket")" ] || fail "no check block: nothing to prove the build against"

# worktree, fresh from the remote
git -C "$REPO" fetch -q "$REMOTE" "$BASE" || fail "fetch failed" retry
cleanup; git -C "$REPO" worktree add -q -b "foundry/$ID" "$WT" "$REMOTE/$BASE" || fail "worktree failed" retry
BASE_SHA=$(git -C "$WT" rev-parse HEAD)

# build
PROMPT=$(fill prompts/build.md)
[ -f "state/$ID.reasons" ] && PROMPT+=$'\n\n## The last attempt failed, because:\n'"$(cat "state/$ID.reasons")"
( cd "$WT" && timeout "$BUILD_SECS" "$CLAUDE" -p "$PROMPT" --permission-mode bypassPermissions ) \
  </dev/null > "logs/$ID.build.log" 2>&1 || log "$ID build exited $?"
git -C "$WT" add -A && git -C "$WT" commit -q -m "$ID: wip at cap" >/dev/null 2>&1

# gate: scripts only, no model
cmp -s "tickets/$ID.md" "state/$ID.ticket" || fail "the ticket was edited during the build"
[ -s "$REPORT" ] || fail "no report" retry
while IFS= read -r line; do
  ( cd "$WT" && bash -o pipefail -c "$line" ) </dev/null >> "logs/$ID.checks.log" 2>&1 || fail "check failed: $line" retry
done < <(checks "state/$ID.ticket")
for f in $(git -C "$WT" diff --name-only --diff-filter=MD "$BASE_SHA" HEAD | grep -E '(^|/)tests?/|_test\.|\.test\.'); do
  grep -qF "$f:" "$REPORT" || fail "test file changed with no reason in the report: $f" retry
done
( cd "$WT" && bash -o pipefail -c "$TEST_CMD" ) </dev/null > "logs/$ID.suite.log" 2>&1 || fail "test suite red before review" retry

# review: a fresh session, a fixed answer shape
SCHEMA='{"type":"object","properties":{"verdict":{"type":"string","enum":["pass","fail"]},"reasons":{"type":"array","items":{"type":"string"}}},"required":["verdict","reasons"]}'
RPROMPT=$(fill prompts/review.md)
( cd "$WT" && timeout "$REVIEW_SECS" "$CLAUDE" -p "$RPROMPT" --output-format json --json-schema "$SCHEMA" ) \
  </dev/null > "logs/$ID.review.json" 2>> "logs/$ID.review.err"
V=$(jq -r '.structured_output.verdict // (.result | try fromjson | .verdict) // empty' "logs/$ID.review.json" 2>/dev/null)
jq -r '(.structured_output.reasons // (.result | try fromjson | .reasons) // [])[]' "logs/$ID.review.json" 2>/dev/null \
  | sed 's/^/- /' > "state/$ID.reasons"
[ "$V" = pass ] || fail "review: ${V:-no verdict}$(printf '\n'; cat "state/$ID.reasons")" retry

# merge the base in, test again, push one commit
git -C "$WT" fetch -q "$REMOTE" "$BASE" || fail "fetch before merge failed" retry
git -C "$WT" merge -q --no-edit "$REMOTE/$BASE" >/dev/null 2>&1 || { git -C "$WT" merge --abort; fail "conflicts with $BASE" retry; }
( cd "$WT" && bash -o pipefail -c "$TEST_CMD" ) </dev/null >> "logs/$ID.suite.log" 2>&1 || fail "suite red after merging $BASE" retry
git -C "$WT" reset -q --soft "$REMOTE/$BASE" && git -C "$WT" commit -q -m "$ID: $TITLE"
git -C "$WT" push -q "$REMOTE" "HEAD:$BASE" || { ATT=0; fail "push refused, $BASE moved: next run retries" retry; }

set_st "$ID" done; rm -f "state/$ID.reasons"; log "$ID done: $(git -C "$WT" rev-parse --short HEAD)"; cleanup
```

What to add next, in this order: the red-at-base lint in your ticket filing step (section 3); a stall watcher; a staging deploy after the push; then the pieces under "Once the basics run", starting with the map and the runaway stop.

---

Made by grotech. We build automation like this for businesses: [grotech.dev](https://grotech.dev)

This guide is licensed under [CC BY 4.0](LICENSE): share and adapt it, with credit.
