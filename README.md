# Lab: Managing LLM Context

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

Everyone will need to use `dic` for this lab.
(I've added debug output to it that `llm` does not support,
and so you can't see the underlying API requests in `llm` that you can in `dic`.)

Even if you already have it installed,
you should get the latest version by running
```
$ pip3 install git+https://github.com/mikeizbicki/dic --upgrade
```

You should also ensure that your groq API key is in the environment,
set the default model to be groq with qwen,
and set the temperature to be 0
(which makes the model a bit more deterministic... but not fully deterministic... for complicated parallelism reasons...)
```
$ export GROQ_API_KEY=...
$ export DIC_MODEL=groq+qwen
$ export DIC_OPTION='["temperature=0"]'
```

## Part 1: OpenAI API protocol

All LLM providers support the OpenAI protocol.
Direct competitors like Anthropic have developed their own alternative protocols,
but they still all support the OpenAI protocol as a fallback.
Both Groq and OpenRouter support the OpenAI protocol natively.

Under the hood, all API requests are just json objects.
Here is a json object asking a simple question "What is my name?" to the `qwen/qwen3.8-27b` model:
```
$ cat > r1.json <<EOF
{
  "model": "qwen/qwen3.8-27b",
  "max_tokens": 100,
  "messages": [
    {"role": "user", "content": "What is my name?"}
  ]
}
EOF
```
We can use the curl command to pass this to the API endpoint like so
```
$ curl -sS https://api.groq.com/openai/v1/chat/completions \
    -H "Authorization: Bearer $GROQ_API_KEY" \
    -H 'Content-Type: application/json' \
    -d @r1.json
```
Recall that curl performs an ordinary web request just like your web browser does and prints the contents to stdout (instead of rendering them in a window).
The `-H` fields above pass in the header information with your API key for authorization,
and the `-d @r1.json` passes in the json file we created above.

Observe that the output is a big JSON object.
Pipe the output to `jq` to make it more readable.
```
$ curl -sS https://api.groq.com/openai/v1/chat/completions \
    -H "Authorization: Bearer $GROQ_API_KEY" \
    -H 'Content-Type: application/json' \
    -d @r1.json | jq
```
The API provides a lot of info in this object.
You can find the actual LLM response at `.choices[0].message.content`.
Observe that the model can't answer your question because your original JSON object didn't contain any info about your name.

A multi-round conversation with an LLM is just an API request where the `messages` list contains all of the previous messages in the conversation.
The API provider has no internal memory of what previous messages you have sent,
and you must explicitly provide them each time.

Observe the output from the following API request where try to follow up, but you do not provide the old conversation history:
```
$ cat > r2.json <<EOF
{
  "model": "qwen/qwen3.8-27b",
  "max_tokens": 100,
  "messages": [
    {"role": "user", "content": "What question did I just ask?"}
  ]
}
EOF
$ curl -sS https://api.groq.com/openai/v1/chat/completions \
    -H "Authorization: Bearer $GROQ_API_KEY" \
    -H 'Content-Type: application/json' \
    -d @r2.json | jq `.choices[0].message.content`
```
Notice that the model cannot answer the question.

But if we re-write the `messages` list to contain our previous conversation history,
then the model will be able to answer the question.
Observe:
```
$ cat > r2b.json <<EOF
{
  "model": "qwen/qwen3.8-27b",
  "messages": [
    {"role": "user", "content": "What is my name?"},
    {"role": "assistant", "content": "I don't have enough information to answer that question."},
    {"role": "user", "content": "What question did I just ask?"}
  ]
}
EOF
$ curl -sS https://api.groq.com/openai/v1/chat/completions \
    -H "Authorization: Bearer $GROQ_API_KEY" \
    -H 'Content-Type: application/json' \
    -d @r2b.json | jq `.choices[0].message.content`
```

> **NOTE:**
> Observe that in the definition of `r2b.json` at field `messages[1].content`,
> the content I inserted there does not match the content that your LLM replied to your curl command.
> Recall that API provides have no memory at all of previous conversations,
> and so you can put in the messages list whatever you want,
> and inject whatever words you want into the "mouth" of the LLM assistant.

Now with a 3rd round of communication,
we can get the model to answer our original question.
```
$ cat > r3.json <<EOF
{
  "model": "qwen/qwen3.8-27b",
  "messages": [
    {"role": "user", "content": "What is my name?"},
    {"role": "assistant", "content": "I don't have enough information to answer that question."},
    {"role": "user", "content": "What question did I just ask?"}
    {"role": "assistant", "content": "You asked about your name, but I don't know your name."},
    {"role": "user", "content": "My name is Bob. Answer the question."}
  ]
}
EOF
$ curl -sS https://api.groq.com/openai/v1/chat/completions \
    -H "Authorization: Bearer $GROQ_API_KEY" \
    -H 'Content-Type: application/json' \
    -d @r3.json | jq `.choices[0].message.content`
```

Building these json objects by hand is obviously tedious,
and that is what the `dic` and `llm` command line programs do for us automatically.

## Part 2: `dic` splices the array

`dic` builds the json object for you.
If you pass the `--print-body` flag it will show you the json without actually sending the API request.
```
$ dic --print-body 'What is my name?'
```
> **NOTE:**
> `dic` adds a few more entries into the json object here,
> but it should be obvious that the "substance" of the message is the same.

The `--print-curl` flag shows you the whole curl command.

```
$ dic --print-curl 'What is my name?'
```

Now let's have the actual conversation:

```
$ dic 'What is my name?'
$ dic -c 'My name is Bob.'
```

The second command sends round 1's messages, plus the assistant's reply,
plus your new words: the same splice you did by hand.

```
$ dic --print-body -c 'What is my name?'
```
Observe that the json object now contains the previous conversation inside of it because we are using the `-c` flag to continue the conversation.

Every round of `-c` re-sends the whole chain, so a `k`-turn thread ships
roughly `k^2/2` turn-sized arrays in total: the cost of a conversation grows
with the square of its length, not with its length. Turn three is cheap;
turn fifty is not. This is the curve the tree in Part 4 exists to break.

Actually sending the API request we get
```
$ dic -c 'What is my name?'
```
And even though we asked the exact same question here as in the first `dic` command,
because our `messages` list contains the previous conversation history,
the model is able to answer it.

If we forget the `-c`,
the model no longer has access to the conversation history and can no longer answer our question.
```
$ dic --print-curl 'What is my name?'
$ dic 'What is my name?'
```

Now that we have started a new conversation (by omitting the `-c`),
adding it back in will not get us back to our previous conversation.
Run the following commands to observe that the messages list no longer contains the name "Bob" and that the LLM cannot answer the question.
```
$ dic -c --print-curl 'What is my name?'
$ dic -c 'What is my name?'
```

To get back to our previous conversation, we need to pass an explicit *message id* via the `--mid` flag.
Observe that every run of `dic` outputs a `--mid=COMPLICATED_HASH` in the final line of output.
By copy/pasting this into a new dic command,
you can continue an old conversation.
Find the `--mid` for your previous "My name is Bob" message above, then run
```
$ dic --mid=<your_mid> --print-curl 'What is my name?'
$ dic --mid=<your_mid> 'What is my name?'
```

The only thing that `-c` does is set the `--mid` to be whatever the previous message id was for your current shell session.
Now if you use `-c`, it will still have access to your name.
```
$ dic -c --print-curl 'What is my name?'
$ dic -c 'What is my name?'
```

## Part 3: the array *is* sqlite rows

`dic` and `llm` both store all of their messages in a sqlite3 database.
You can view the schema with the command
```
$ sqlite3 ~/.config/fac/dic.db '.schema messages'
```

There is a lot of debug information stored in this table.
But the important columns are `mid`, `prev_mid`, `user`, and `assistant`.
`prev_mid` points to the previous message idea,
and forms a *linked list* in the `messages` table.
You can observe the whole linked list of your conversation with the command
```
$ sqlite3 ~/.config/fac/dic.db <<EOF
.mode markdown
SELECT mid, prev_mid, user, response FROM messages;
EOF
```

`dic` runs sql queries like this to perform all of its operations.
The following query gives a summary of the runtimes of all llm providers that you've ever used by doing a different SQL query.
```
$ dic --stats
```

`ms_ttft` measures prefill, not decode: it is the time before the first token
appears, and prefill compares every token in the array against every other
token. Time-to-first-token therefore grows with the array while per-token
decode time stays roughly flat, so a long context makes a model feel slow at
the start of a reply and not in the middle of it.


## Part 4: context is a tree

`prev_mid` is a pointer, and two rows cannot share the same `prev_mid` forming a tree structure.
Find the mid of the conversation you have been having:

Now fork it in two directions:

```
$ dic --mid=<mid> 'I lied my name is Carl'
<...> --mid=<mid1>
$ dic --mid=<mid> 'I lied my name is Denise'
<...> --mid=<mid2>
```

Both new rows have `prev_mid = $M`. You did not overwrite the conversation;
you grew a second branch off it, and the first one is still there.

```
$ dic --mid=<mid1> 'What is my real name?'
$ dic --mid=<mid2> 'What is my real name?'
```

Compaction is not a flag; the tree *is* the mechanism. Ask the old thread to
summarize itself (`dic -c 'summarize our decisions in 200 words.'`), then
start a fresh root with `dic -s "$S" 'continue'`, where `$S` is that summary.
The old chain is not deleted, and it stays one `--mid` away.


## Part 5: commit on `example-median`

Now the real thing.
`example-median` is a small python module with a bug.

First confirm the bug:
```
$ cd example-median
$ python3 -m pytest
$ python3 -c 'import stats; print(stats.median([1,2,3,4]))'
```

### 5a: investigate with `dic`

Recall that `files-to-prompt` is a `cat` that prints file names and skips hidden files.
Give the model the whole repo, then ask:

```
$ dic <<EOF
$(files-to-prompt . .github)
Are there any bugs in this repo?
EOF
```

Now ask for the fix:
```
$ dic -c 'What other python libraries could I use for the median besides this one?'
```

### 5b: commit with `committe`

`committe` is the coding agent you built in the previous lab. It reads the
conversation, asks for a patch, and applies it:

```
$ committe -c 'apply that fix'
```

<!--
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
-->
