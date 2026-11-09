---
name: trio
description: Run the same task with Claude, Codex and Gemini (agy) independently, compare the three results, settle every disagreement with evidence and the advisor, and give one final answer.
argument-hint: [task]
disable-model-invocation: true
allowed-tools: Bash(codex exec *) Bash(codex --version) Bash(agy -p *) Bash(agy --version) Bash(git worktree *) Bash(git rev-parse *) Bash(git status *) Bash(git diff *)
---

Do this task three times: yourself, with Codex (`codex exec`), and with
Gemini (`agy -p`). Then compare the three results and settle every difference
before giving a final answer.

TASK: $ARGUMENTS

## Rules for the whole run
- Judge by evidence, never by which model wrote an answer. Two models agreeing
  is a hint about where to look first, never proof.
- Do not open Codex's or Gemini's output until your own version is finished.
- Shell variables do not carry over between Bash commands. Always write full
  absolute paths.
- Never retry a quota or location error in a loop.

## If a model fails
A model has failed when its command exits non-zero; when Gemini's JSON has a
`status` other than SUCCESS; when its stderr mentions location, quota,
RESOURCE_EXHAUSTED, authentication or workspace trust; or, in CODE mode, when
its worktree has no changes and its stderr shows denied actions.
Stop and use AskUserQuestion: show the error and offer (a) retry once,
(b) continue without that model, (c) cancel the run. Let the other background
job keep running while you ask. If Gemini's error mentions workspace trust,
tell me to run `agy` once interactively in that folder and choose trust.
Whatever happens, the report says which models actually ran.

## 1. Preflight (cheap, before any real work)
- Run `codex --version` and `agy --version`.
- One tiny Gemini call to confirm sign-in and location, from the current folder:
  `agy -p "Reply with: ok" --output-format json --print-timeout 2m < /dev/null`
  If it fails, follow "If a model fails" now, before creating anything.

## 2. Set up
- Pick a timestamp T in the form YYYYMMDD-HHMM. Create ~/trio-runs/T/.
  Below, RUN means that folder's full absolute path.
- Model notes at the start of the task text are removed from the task and
  passed as flags. Otherwise add no model flags; each tool uses its saved default.
  - "(Codex: <model>, effort <level>)" -> `-m <model> -c model_reasoning_effort=<level>`
  - "(Gemini: <model>, effort <level>)" -> `--model <model> --effort <level>`
- Save the task text (without those notes) to RUN/task.md.
- Choose the mode:
  - CODE: the task changes files in this git repo.
  - ANSWER: anything else (question, derivation, writing, trip guide,
    trip report).

## 3. Start Codex and Gemini in the background (two Bash calls with run_in_background)
Always end the Gemini command with `< /dev/null`: agy reads stdin, and without
a terminal it would wait forever.
- ANSWER (both are read-only here: they can read files but not change them):
  - Codex: `codex exec -C <abs current folder> --skip-git-repo-check -o RUN/codex.md "$(cat RUN/task.md)"`
  - Gemini: `cd <abs current folder> && agy -p "$(cat RUN/task.md)" --output-format json --print-timeout 30m < /dev/null > RUN/gemini.json 2> RUN/gemini.err`
- CODE:
  - Note the base commit: `git rev-parse HEAD`. Below, BASE means that commit
    and REPO means the current folder's name. Create three worktrees:
    `git worktree add ../REPO-trio-T-claude -b trio/T-claude BASE`
    `git worktree add ../REPO-trio-T-codex -b trio/T-codex BASE`
    `git worktree add ../REPO-trio-T-gemini -b trio/T-gemini BASE`
  - Codex: `codex exec -C <abs path of ../REPO-trio-T-codex> --sandbox workspace-write -o RUN/codex.md "$(cat RUN/task.md) Work only in this folder. Do not commit."`
  - Gemini: `cd <abs path of ../REPO-trio-T-gemini> && agy -p "$(cat RUN/task.md) Work only in this folder. Do not commit." --sandbox --mode auto --output-format json --print-timeout 30m < /dev/null > RUN/gemini.json 2> RUN/gemini.err`
- If the task depends on current facts (opening hours, prices, recent papers),
  add `--search` to the Codex command (drop it if your Codex rejects it).
  Gemini has web access by default.

## 4. Do the task yourself while they run
- CODE: work only in ../REPO-trio-T-claude. Do not commit.
- ANSWER: write your complete answer to RUN/claude.md.
- Then wait for both background jobs. If one is still running 20 minutes after
  you finished, ask me whether to keep waiting.
- When Gemini finishes: check that `jq -r .status RUN/gemini.json` prints
  SUCCESS, then `jq -r .response RUN/gemini.json > RUN/gemini.md`. Read
  RUN/gemini.err too. If jq is missing, use
  `python3 -c 'import json,sys; print(json.load(open(sys.argv[1]))["response"])' RUN/gemini.json`.

## 5. Compare
- Read RUN/codex.md and RUN/gemini.md. In CODE mode, in each worktree run
  `git status`, save `git diff BASE > RUN/<claude|codex|gemini>.diff`, and run
  the tests.
- Write a numbered list of every point where the results differ, with what
  Claude, Codex and Gemini each say: a different result, step, claim, fact,
  number, design choice, test outcome, or something one included and the
  others left out. Mark 2-against-1 splits and check the odd one out first,
  but check every point.
- If there are no differences at all: skip step 6. Still do one independent
  check of the key result, because three models can share a mistake.

## 6. Settle each difference (only when there are differences)
a. Evidence first. For each numbered point, check it yourself: redo the
   derivation step, run a quick calculation or script, run the tests, or
   check a source. Note what you found under that point.
b. CODE mode cross-review, each read-only, in rotation:
   - Codex reviews your worktree:
     `codex exec -C <abs path of ../REPO-trio-T-claude> "Review the uncommitted changes in this folder against commit BASE. List bugs and weak points. Do not edit files."`
   - Gemini reviews Codex's diff from the saved file (it cannot run git in
     read-only mode):
     `cd RUN && agy -p "Review the diff in codex.diff. List bugs and weak points." --output-format json --print-timeout 15m < /dev/null | jq -r .response`
   - You review Gemini's worktree yourself.
   Add the findings to the list.
c. Put everything in the conversation. The advisor sees only this conversation,
   not files on disk. Use the Read tool on RUN/task.md, RUN/claude.md,
   RUN/codex.md and RUN/gemini.md (in CODE mode, the three diffs), then write
   out the numbered differences with the three positions and the evidence for
   each, ending with:
   "Advisor: for each numbered point, which version is right, and does the
   evidence support it?"
d. Call the advisor tool once. It takes no input; it reads what you just wrote.
e. If the advisor's verdict conflicts with your evidence on any point, do not
   switch silently. Write "On point N I found X, you said Y. Which constraint
   breaks the tie?" and call the advisor once more.
f. If a point is still unclear after (e), stop and use AskUserQuestion: show
   the three versions, the evidence and the advisor's view, and let me choose.
   Then continue.
- If there is no advisor tool in this session, say so in the report, rely on
  (a) and (b), and tell me to run `/advisor fable`.

## 7. Report
Write RUN/compare.md and show it in chat:
- Header: task, mode, which of the three models ran, and the model and effort
  each one used.
- Table with one row per difference:
  # | point | Claude | Codex | Gemini | verdict | evidence | advisor agrees?
- CODE: test results for each worktree, cross-review findings, and which
  branch you recommend and why.
- ANSWER: the final combined answer. Mark any part that I chose in step 6f.
- One line on what came from each model.

## 8. Stop and ask
Do not merge branches, delete worktrees, or change the original files.
Use AskUserQuestion to ask me what to keep:
- CODE: keep one branch or combine them; and whether to remove the worktrees now.
- ANSWER: save the final answer to RUN/final.md, or revise it first.
