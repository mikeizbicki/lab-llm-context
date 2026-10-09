# Lab: the array is the mind

A chat model has no memory.

Every time you "continue a conversation" with an LLM API, the client sends
the *entire* conversation again, as an array of messages. The model does not
remember the previous turn; it reads it a second time. Everything that makes
a chat model feel like a conversation is therefore a property of that array:
what is in it, who put it there, and what it cost.

In this lab you will take that array apart by hand, watch `dic` build it for
you, find it sitting in a sqlite table, discover that it is a *tree* and not
a list, and then aim the whole apparatus at a broken python module in the
`example-median` submodule.

The rest of the course uses `dic`. If you have not done the `dic` lab yet,
do that one first, or at least read it: this lab assumes you know what `-c`,
`--mid`, `-m`, `-s` and `-x` do.

## Setup

Put a groq key in your environment, and settle the two defaults this lab
uses everywhere:

```
$ export GROQ_API_KEY=...
$ export DIC_MODEL=groq+qwen
$ export DIC_OPTION='["temperature=0"]'
$ dic --models
```

`temperature=0` makes the model as deterministic as a language model can be,
which matters when thirty people in one room are supposed to see the same
answer.

> **NOTE:**
> `DIC_MODEL` applies only to a *new* conversation. Once a thread exists,
> `-c` and `--mid` inherit the model the thread last used, so an exported
> default can never change the provider of a running conversation halfway
> through. That is the same "new conversations only" rule that `-s` follows,
> and it is the reason an exported default is safe.

Then fetch the submodule, which is a git repo inside this git repo:

```
$ git submodule update --init --recursive
$ ls example-median
```

It ships with `stats.py`, `tests/test_stats.py`, and a `README.md` that
contains a small surprise you will meet in Part 5.

## Part 1: three rounds, raw curl

Start at the bottom, with no tools at all.

Ask the model a question it cannot answer:

```
$ cat > r1.json <<EOF
{
  "model": "qwen/qwen3.8-27b",
  "messages": [
    {"role": "user", "content": "What is my name?"}
  ]
}
EOF
```

and send it with `curl`, exactly as the API documentation describes:

```
$ curl -sS https://api.groq.com/openai/v1/chat/completions \
    -H "Authorization: Bearer $GROQ_API_KEY" \
    -H 'Content-Type: application/json' \
    -d @r1.json
```

The model says it does not have enough information, which is correct:
nothing in that array says your name.

Now write `r2.json`. It is `r1.json`, plus the assistant reply from round 1,
plus a new user message:

```
$ cat > r2.json <<EOF
{
  "model": "qwen/qwen3.8-27b",
  "messages": [
    {"role": "user", "content": "What is my name?"},
    {"role": "assistant", "content": "<paste round 1's reply here>"},
    {"role": "user", "content": "My name is Bob."}
  ]
}
EOF
```

Then `r3.json` is `r2.json`, plus round 2's reply, plus the question again.
Five messages, two of them assistant turns.

```
$ curl -sS https://api.groq.com/openai/v1/chat/completions \
    -H "Authorization: Bearer $GROQ_API_KEY" \
    -H 'Content-Type: application/json' \
    -d @r3.json
```

The answer is `Bob.`, because *you* put it in the array.

> **NOTE:**
> Rounds 1 and 3 end with the exact same question and get two different
> answers. The difference is not in the model. The model saw the same
> question both times; it saw a different array.

> **OPTIONAL:**
> If you would rather not use names, pick a fact that arrives early and is
> asked about later. My favorite is `Alice is taller than Bob. Bob is
> taller than Carol. Who is tallest?`: ask only the last sentence, then ask
> it again after adding the first two. Same lesson, and the answer is not a
> string you typed.

This is the whole mechanism, and it is already tedious: five messages by
hand for three rounds, so a conversation of n rounds costs O(n) hand-written
messages. Nobody does five rounds by hand, which is why nobody can see what
a chat client is really sending.

## Part 2: `dic` splices the array

`dic` builds the array for you, and will show you the array it built without
sending anything.

```
$ dic --print-body 'What is my name?'
```

You get round 1's JSON back: one message, the same one you wrote by hand.
`--print-curl` prints the same request as a shell command you can paste:

```
$ dic --print-curl 'What is my name?'
```

Both stop inside `client.py` before the network is touched
(`if printing: ... return Reply`), so nothing is sent and nothing is billed.

Now have the actual conversation:

```
$ dic 'What is my name?'
$ dic -c 'My name is Bob.'
```

The second command sends round 1's messages, plus the assistant's reply,
plus your new words: the same splice you did by hand.

```
$ dic --print-body -c 'What is my name?'
```

prints round 3. Compare it against your `r3.json`: it is the same array,
reassembled from the database.

Two useful incantations while you are here:

```
$ dic -v  'hello'    # the timings of the call
$ dic -vv 'hello'    # and the POST body, echoed before it is sent
```

The cost line ends with `--mid=01ABC…`, which is the primary key of the row
this turn was written to. Copy it somewhere.

> **NOTE:**
> `-c` is pure sugar for
> `--mid $(cat $XDG_RUNTIME_DIR/fac/dic/$DIC_SESSION)`. It continues *this
> shell's* conversation and not a global one: two terminals get two
> conversations, and a subshell inherits the same one.

## Part 3: the array *is* sqlite rows

Look at what `dic` wrote:

```
$ sqlite3 ~/.config/fac/dic.db '.schema messages'
```

Every column is there. The one that matters now is `prev_mid`: it points at
the previous message, and a walk of it is the conversation. That walk is
`store.history`, in `dic/store.py`, and it is one recursive CTE over
`prev_mid` returning the rows reversed. That function *is* the messages
array you typed out by hand in Part 1.

```
$ sqlite3 ~/.config/fac/dic.db \
    "SELECT mid, prev_mid, substr(user,1,30) FROM messages ORDER BY t_start DESC LIMIT 5;"
```

Look next at `store.SCHEMA`'s `stats` view. It is a `CREATE VIEW`: nothing in
it is stored. It says, for instance,

- `ms_ttft = (t_first - t_start) / 1e6`
- `head = substr(model_id, 1, instr(model_id || '+', '+') - 1)`

Every duration in `dic` is the *subtraction of two stored instants*. `dic`
never stores a duration and so can never store an inconsistent one, and
sqlite does the arithmetic instead of python.

So each readout command is one `GROUP BY` over that view, and python only
joins cells with tabs:

```
$ dic --stats
$ dic --providers
$ dic --cost-of HEAD~3
```

No server, one file, ACID, readable by `sqlite3` as happily as by `dic`.

> **NOTE:**
> `--stats` is shaped for `awk` and `sort` on purpose. Try
> `dic --stats | sort -t$'\t' -k4 -n` and see what you get.

## Part 4: context is a tree

`prev_mid` is a pointer, not a stack, and nothing says two rows cannot point
at the same place. Find the mid of the conversation you have been having:

```
$ M=$(cat "$XDG_RUNTIME_DIR/fac/dic/${DIC_SESSION:-global}")
$ echo $M
```

Now fork it in two directions:

```
$ dic --mid=$M 'A'
$ dic --mid=$M 'B'
```

Both new rows have `prev_mid = $M`. You did not overwrite the conversation;
you grew a second branch off it, and the first one is still there.

```
$ dic --log --graph
```

draws the forest, one column per conversation still being walked, with
`|` `/` `*` exactly as `git log --graph` draws it.

A message can be named three ways, all resolved by `store.resolve_ref`:

```
$ dic --show HEAD      # this session's last message
$ dic --show HEAD~3    # three steps up its ancestor chain
$ dic --show 01ABC     # any unambiguous ULID prefix
```

An ambiguous prefix is an error, never a guess.

**Sessions.** A session pointer is one file in tmpfs, named after
`DIC_SESSION` with `%` and `/` percent-escaped. A `/` therefore makes a
*sub-session*, and the cost of a whole run is one `LIKE 'parent/%'` over the
`session` column:

```
$ export DIC_SESSION=p
$ dic -c 'keep going'
$ dic --log --session p     # every message in p and p/...
$ dic --cost-session p      # what that subtree cost
$ dic --cost-tree p         # and its per-session breakdown
```

Two terminals with two sessions write to the one `messages` table with no
coordination beyond sqlite's file lock. "Isolation" costs nothing because it
was never isolation: it is a naming convention.

**Compaction is not a flag.** The tree is the mechanism. When a conversation
has grown too long to be worth sending, summarize it and start a fresh root
whose system prompt *is* the summary:

```
$ S=$(dic -c 'summarize our decisions in 200 words.')
$ dic -s "$S" 'continue'
```

`-s` applies only when a conversation is *new* (`client.py`: `elif system is
None`), so an exported `DIC_SYSTEM` can never rewrite the system prompt of a
running thread. The old chain is not deleted. It is one `--mid` away.

## Part 5: commit on `example-median`

Now the real thing. `example-median` is a small python module with a bug, a
test suite that catches it, and a README you should read before you trust
anything I say about it.

First confirm the bug:

```
$ cd example-median
$ python3 -m pytest
$ python3 -c 'import stats; print(stats.median([1,2,3,4]))'
```

`3` is not the median of `[1,2,3,4]`. Look at `stats.py`:

```python
ordered = sorted(data)
return ordered[len(ordered) // 2]
```

That line reads as obviously correct, which is exactly why you need the
test: only the *test* knows that an even-length sequence has two middles.
The fix is five lines.

### 5a: investigate with `dic`

`files-to-prompt` is a `cat` that prints file names and skips hidden files.
Give the model the whole repo, then ask:

```
$ dic <<EOF
$(files-to-prompt . .github)
Which test is failing and why? Answer in one sentence.
EOF
```

If you got a refusal instead of an answer, good. Read
`example-median/README.md`, find the *prompt injection*, and understand what
happened: the model was handed your instruction and my instruction in the
same array, and it could not tell which was instruction and which was data.
This is the lesson from Part 1, one level down. Instructions and data are
both "the array is the mind."

Now ask for the fix, and pick up the `--mid=…` that `dic` printed on stderr:

```
$ dic -c 'What is the minimal fix? Show only the diff.'
$ export M=01ABC…    # the mid from the line above
```

### 5b: commit with `committe`

`committe` is the coding agent you built in the previous lab. It reads the
conversation, asks for a patch, and applies it:

```
$ committe -c 'apply that fix'
```

Three exit codes are worth knowing: `0` means committed, `1` means the patch
was unusable, and `2` means the model asked a question instead of answering
and there was nothing to apply.

The patch is always at

```
$ cat "$(git rev-parse --git-dir)/committe-patchfile"
```

and that is one of the files you will submit.

> **NOTE:**
> `committe-mkpatch` passes its instructions with `--prepend-prompt`, which
> puts them in the *user* turn, which means they do not invalidate the
> cached prompt prefix. Putting them in `-s` would rewrite the head of the
> conversation and throw the cache away. Prefixes are the whole reason a
> cache exists.

> **NOTE:**
> The `committe.sh` you wrote by hand is a teaching version. The production
> one adds `-f`, `--retries` and `COMMITTE_RETRIES`, and your shell startup
> file keeps it with `eval "$(dic --init)"`.

### 5c: fork at `M`

Here is what the tree is for. Go back to the mid you saved and ask for the
fix a second way:

```
$ dic --mid=$M 'fix using statistics.median'
$ dic --mid=$M 'fix without importing anything'
```

Two branches, two different fixes, both rooted at the same investigation.
Write each patch out with `--path`, or let `committe -c 'apply'` stage each
on its own branch, and run the tests against each:

```
$ python3 -m pytest
```

```
$ dic --log --graph
```

shows the fork. Keep the winner, discard the other, or don't: the losing
branch is not deleted, and it is one `--mid` away if you change your mind.
*This* is why the tree exists.

### 5d: a parallel session

Open a second terminal and export a different session name:

```
$ export DIC_SESSION=fixB
```

Redo the investigation from scratch. You will get a different patch, because
the model is nondeterministic and the context is different, and because
neither of you is looking at the other's array.

Now compare what the two attempts cost:

```
$ dic --cost-session fix
$ dic --cost-session fixB
$ dic --cost-tree fix
```

The `session` column is what makes that one query per attempt, and
`--cost-tree` breaks the subtree down session by session.

## Submission

In the `example-median` submodule, push three branches:

- `fix`, branched from `master`, containing the fix you kept
- `fork-a`, the first alternative fix
- `fork-b`, the second alternative fix

Also submit:

- `$(git rev-parse --git-dir)/committe-patchfile` from the winning commit
- one line of `dic --stats`

> **BONUS:**
> `dic --log --graph --session fix` for the whole run, rendered as an image.
> `--graph` already emits the columns; graphviz draws them.

> **NOTE:**
> Finish by running `python3 -m pytest` in `example-median` from a clean
> checkout of your `fix` branch. A patch file is worth nothing if it does
> not apply to the branch it names.
