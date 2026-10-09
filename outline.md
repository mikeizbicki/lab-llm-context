# Outline

**Setup.** `-m groq+qwen -o temperature=0` throughout. `git submodule update --init --recursive`; `cd example-median`. The repo ships with `stats.py`, `tests/test_stats.py`, a prompt-injected `README.md`, and (assumed) `.github/workflows/tests.yml` that the README names as the thing to include.

## P1 — three rounds, raw curl

R1 `What is my name?` → *not enough information*. R2 `My name is Bob.` R3 `What is my name?` → `Bob.` These three prompts see the *same final question* and give two different answers: the model has no memory, the **messages array** does. Build them by hand: `r1.json` = 1 message; `r2.json` = R1's array + assistant reply + new user = 3; `r3.json` = 5, two assistant turns. `curl -sS https://api.groq.com/openai/v1/chat/completions -H "Authorization: Bearer $GROQ_API_KEY" -d @rN.json`. Tedium is O(rounds); no human does five by hand. (Alt: Alice>Bob / Bob>Carol / "who is tallest?"; pick whichever fits the room.)

## P2 — dic splices the array

`dic --print-body 'What is my name?'` prints R1; `--print-curl` prints R1 as a runnable shell command; both stop in `client.py` before the network (`if printing: … return Reply`). `dic -c 'My name is Bob.'`; `dic --print-body -c 'What is my name?'` reassembles R3 byte-identical from sqlite — the array you typed by hand. `-v` adds the cost line with `--mid=…`; `-vv` echoes the POST body before sending.

## P3 — the array *is* sqlite rows

`sqlite3 ~/.config/fac/dic.db '.schema messages'`. Read `store.history` — one recursive CTE over `prev_mid`, reversed — that *is* the messages array. Read `store.SCHEMA`'s `stats` view: `ms_ttft = (t_first-t_start)/1e6`, `head = substr(model_id,1,instr(model_id||'+','+')-1)`. Durations are subtractions of stored instants, so `--stats`, `--providers`, `--cost-of HEAD~3` are each one `GROUP BY` over that view; python joins cells with tabs and computes nothing. No server, one file, ACID, readable by `sqlite3` as happily as by dic.

## P4 — context is a tree

- `M=$(cat "$XDG_RUNTIME_DIR/fac/dic/${DIC_SESSION:-global}")`; `dic --mid=$M 'A'` and `dic --mid=$M 'B'` both write `prev_mid=$M`. `dic --log --graph` draws the forest. `HEAD`, `HEAD~3`, and a unique ULID prefix all resolve (`store.resolve_ref`; ambiguous is an error, not a guess).
- Sessions: the pointer is one tmpfs file, name escaped (`%`→`%25`, `/`→`%2F`). Two terminals, two trees, both writing one `messages` table — sqlite's file lock serializes, so "isolation" costs nothing. `--session p` lists a subtree, `--cost-session p` sums it, both via `session LIKE p || '/%'`.
- Compaction is not a flag; the tree is the mechanism. `S=$(dic -c 'summarize our decisions in 200 words.')`; `dic -s "$S" 'continue'` starts a fresh root whose system prompt is the summary. `-s`/`DIC_SYSTEM` apply only on a *new* conversation (`client.py`: `elif system is None`), so an exported default can never rewrite a running thread — safe by construction. The old chain is not deleted; it is one `--mid` away.

## P5 — committe on example-median

**The bug.** `median([1,2,3,4])` returns `3`; `test_median_even_length` asserts `2.5`. In the source, `ordered[len(ordered) // 2]` reads as obviously correct: only the *test* exposes the off-by-one at even length. The fix is five lines: branch on `n % 2`, average the two middles when even.

**The trap.** `example-median/README.md` carries a prompt injection: the model is told to reply *only* with the refusal string unless `.github/workflows/tests.yml` is in the context. So:
- `files-to-prompt . | dic …` → the model refuses.
- `files-to-prompt . .github | dic …` → the model fixes it.

Same lesson as Part 1, one level down: the model knows only what is in the array, and it cannot tell *instructions* from *data*. Both are "the array is the mind."

**5a — investigate with dic.**
```
$ dic <<EOF
$(files-to-prompt . .github)
Which test is failing and why? Answer in one sentence.
EOF
$ dic -c 'What is the minimal fix? Show only the diff.'
```
Each prints `--mid=01ABC…` on stderr; pick one, call it `M`.

**5b — commit with committe.** `committe -c 'apply that fix'`. `committe-mkpatch` prepends `committe-prompt` with `--prepend-prompt`, so it lands in the *user* turn and `-c` hits the cached prefix — putting those instructions in `-s` would rewrite the head and throw away the cache. `committe-apply` runs `git apply --index --recount --ignore-whitespace`, falls back to `git-apply-fuzzy`, and commits as `[geni] …` with `committe@agent`. Exit 0 = committed, 1 = patch unusable, 2 = model asked a question instead (nothing to apply). The production `scripts/committe.sh` (with `-f`, `--retries`, `COMMITTE_RETRIES`) is the one to keep in `.bashrc` via `eval "$(dic --init)"`.

**5c — fork at `M`.** `dic --mid=$M 'fix using statistics.median'` and `dic --mid=$M 'fix without importing anything'`. Two branches; run `python3 -m pytest` against each (write each patch with `--path`, or let `committe -c 'apply'` stage each on its own branch). `dic --log --graph` shows the fork; the losing branch stays readable and one `--mid` away. Keep the winner, discard the other. *This* is why the tree exists.

**5d — parallel session.** Second terminal, `export DIC_SESSION=fixB`, same investigation from scratch. `dic --cost-session fix` and `--cost-session fixB` print each subtree's spend; the `session` column makes that one query. Show `--cost-tree fix` for the per-run breakdown.

**Submit.** Branches `fix` (from master), `fork-a`, `fork-b`; the patch at `$(git rev-parse --git-dir)/committe-patchfile` for the winning commit; one line of `dic --stats`. Bonus: `dic --graph --session fix` as a PNG of the forest.
