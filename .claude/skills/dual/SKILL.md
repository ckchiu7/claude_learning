---
name: dual
description: Run the same task with Claude and Codex independently, compare the two results, settle every disagreement with evidence and the advisor, and give one final answer.
argument-hint: [task]
disable-model-invocation: true
allowed-tools: Bash(codex exec *) Bash(git worktree *) Bash(git rev-parse *) Bash(git status *) Bash(git diff *)
---

Do this task twice, once yourself and once with Codex. Then compare the two
results and settle every difference before giving a final answer.

TASK: $ARGUMENTS

## Rules for the whole run
- Judge by evidence, never by which model wrote an answer. Your own answer is
  not the default and Codex's is not the default.
- Do not open Codex's output until your own version is finished.
- Shell variables do not carry over between Bash commands. Always write full
  absolute paths.
- If Codex fails (not installed, not signed in, sandbox error, no network),
  say so right away, finish with your own answer, and mark the report
  "Codex did not run". Do not pretend the comparison happened.

## 1. Set up
- Pick a timestamp T in the form YYYYMMDD-HHMM. Create ~/dual-runs/T/.
  Below, RUN means that folder's full absolute path.
- If the task text starts with a note like "(Codex: gpt-6-astra, effort xhigh)",
  remove it from the task text and pass it to Codex as
  `-m <model> -c model_reasoning_effort=<effort>`. Otherwise add no model flags.
- Save the task text exactly as given (without that note) to RUN/task.md.
- Choose the mode:
  - CODE: the task changes files in this git repo.
  - ANSWER: anything else (question, derivation, writing, trip guide,
    trip report).

## 2. Start Codex in the background (Bash tool with run_in_background)
- CODE:
  - Note the base commit: `git rev-parse HEAD`. Below, BASE means that commit
    and REPO means the current folder's name.
  - Create two worktrees from it:
    `git worktree add ../REPO-dual-T-claude -b dual/T-claude BASE`
    `git worktree add ../REPO-dual-T-codex -b dual/T-codex BASE`
  - Start Codex in its worktree (absolute path after -C):
    `codex exec -C <abs path of ../REPO-dual-T-codex> --sandbox workspace-write -o RUN/codex.md "$(cat RUN/task.md) Work only in this folder. Do not commit."`
- ANSWER:
  - `codex exec -C <abs path of current folder> --skip-git-repo-check -o RUN/codex.md "$(cat RUN/task.md)"`
  - Codex is read-only by default here: it can read files but not change them.
- If the task depends on current facts (opening hours, prices, recent papers),
  add `--search` for live web results. If your Codex version rejects that flag,
  drop it; Codex still has cached web search by default.

## 3. Do the task yourself while Codex runs
- CODE: work only in ../REPO-dual-T-claude. Do not commit.
- ANSWER: write your complete answer to RUN/claude.md.
- When you are done, wait for the Codex background task to finish. If it is
  still running 20 minutes after you finished, ask me whether to keep waiting.

## 4. Compare
- Read RUN/codex.md. In CODE mode also get both diffs:
  `git diff BASE` inside each worktree, and run the tests in each worktree.
- Write a numbered list of every point where the two results differ: a
  different result, step, claim, fact, number, design choice, test outcome,
  or something one included and the other left out.
- If there are no differences: skip step 5. Still do one quick independent
  check of the key result (redo the main step, or run the tests), because
  both could be wrong in the same way.

## 5. Settle each difference (only when there are differences)
a. Evidence first. For each numbered point, check it yourself: redo the
   derivation step, run a quick calculation or script, run the tests, or
   check a source. Note what you found under that point.
b. CODE mode cross-review. Have Codex review your work read-only:
   `codex exec -C <abs path of ../REPO-dual-T-claude> "Review the uncommitted changes in this folder against commit BASE. List bugs and weak points. Do not edit files."`
   Review Codex's worktree yourself the same way. Add findings to the list.
c. Put everything in the conversation. The advisor sees only this conversation,
   not files on disk. Use the Read tool on RUN/task.md, RUN/claude.md and
   RUN/codex.md (in CODE mode, show both diffs), then write out the numbered
   differences with the evidence for each, ending with:
   "Advisor: for each numbered point, which version is right, and does the
   evidence support it?"
d. Call the advisor tool once. It takes no input; it reads what you just wrote.
e. If the advisor's verdict conflicts with your evidence on any point, do not
   switch silently. Write "On point N I found X, you said Y. Which constraint
   breaks the tie?" and call the advisor once more.
f. If a point is still unclear after (e), stop and use AskUserQuestion: show
   both versions, the evidence and the advisor's view, and let me choose.
   Then continue.
- If there is no advisor tool in this session, say so in the report, rely on
  (a) and (b), and tell me to run `/advisor fable`.

## 6. Report
Write RUN/compare.md and show it in chat:
- Header: task, mode, Codex model and effort used, whether Codex finished.
- Table with one row per difference:
  # | point | Claude | Codex | verdict | evidence | advisor agrees?
- CODE: test results for both worktrees, cross-review findings, and which
  branch you recommend and why.
- ANSWER: the final combined answer. Mark any part that I chose in step 5f.
- One line on what came from each model.

## 7. Stop and ask
Do not merge branches, delete worktrees, or change the original files.
Use AskUserQuestion to ask me what to keep:
- CODE: keep Claude's branch, keep Codex's branch, or combine; and whether to
  remove the worktrees now.
- ANSWER: save the final answer to RUN/final.md, or revise it first.
