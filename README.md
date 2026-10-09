# Lab: Managing LLM Context

<img align=right src=img/meme.webp width=300px />

This lab will teach you how to "manage your context window" using AI.
You will learn:
1. how LLM APIs work under the hood,
2. how almost all AI agents (including `dic`, `llm`, copilot, VSCode plugins, etc) use sqlite3 to store conversation histories, and
3. how to manage these session histories to be more efficient.
The lab focuses on using `dic`, but the techniques generalize to any agent you might use in the future.

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

> **NOTE ON COSTS:**
> Every round of `-c` re-sends the whole chain.
> On the $k$th round of a conversation, we are sending $k-1$ messages,
> so the total number of tokens used in a $k$ round conversation is $\Theta(k^2)$.
> A 50 round conversation (even of short questions/replies) is therefore quite expensive.

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

> **NOTE:**
> The quality of LLM providers is often measured by their *time to first token* (TTFT).
> The average TTFT is shown in the table above,
> and `dic` prints a TTFT counter on every query where the TTFT exceeds 0.5 seconds.
> Due to the way parallelism in LLMs works,
> the TTFT basically doesn't depend on the size of your input JSON.
> The rate that tokens are generated, however, will shrink quadratically as the number of output tokens increases.


## Part 4: context is a tree

`prev_mid` is a pointer, and two rows cannot share the same `prev_mid` forming a tree structure.
Let's rerun the conversation below to generate a new `--mid`.
```
$ dic 'What is my name?'
$ dic -c 'My name is bob'
$ dic -c 'What is my name?'
<...> --mid=<mid>
```
Remember that last `--mid=<mid>` output.

Now fork the conversation in two directions:
```
$ dic --mid=<mid> 'I lied my name is Carl'
<...> --mid=<mid1>
$ dic --mid=<mid> 'I lied my name is Denise'
<...> --mid=<mid2>
```

Both new rows have `prev_mid = <mid>`.
You did not overwrite the conversation;
you grew a second branch off it, and the first one is still there.
You can observer that by running the commands below.

```
$ dic --mid=<mid1> 'What is my real name?'
$ dic --mid=<mid2> 'What is my real name?'
```
You should see "Carl" in the output of the first command and "Denise" in the output of the second.

Managing trees of conversations like this with AI agents is a key skill in controlling your context, no matter what AI system you are working with.

<!--
### Compaction

Sometimes conversations get too large.
*Compaction* is the process of shrinking those conversations.


Reuse the Bob conversation from Part 2. First ask it to summarize itself:

```
$ dic -c 'summarize this conversation in one sentence.'
The user's name is Bob.
--mid=01K4Z9...
```

The response and the trailing `--mid=` line both go to stdout, so capture
the summary by dropping that line:

```
$ S=$(dic -c 'summarize this conversation in one sentence.' | grep -v '^--mid=')
$ echo "$S"
The user's name is Bob.
```

Now start a fresh conversation with `-s`, which sets the system prompt:

```
$ dic -s "$S" 'What is my name?'
Bob.
```

Compare the arrays the two approaches send:

```
$ dic --print-body -s "$S" 'What is my name?' | wc -c
$ dic --print-body -c 'What is my name?' | wc -c
```

Both get the answer right; the second array has to re-read every round of
the original conversation to do it, and it grows every time you add a turn.
`-s` applies only when a conversation *starts*, so a default system prompt
can never rewrite a thread that is already running.

The old chain is not deleted. The summary is a message like any other, and
the full conversation is still one `--mid` away.
-->

## Part 5: commit on `example-median`

Now the real thing.
`example-median` is a small python module with a bug.

First get the code
```
$ git clone https://github.com/mikeizbicki/lab-llm-context
$ cd lab-llm-context
$ git submodule init
$ git submodule update
$ cd example-median
```
Then observe the bug
```
$ python3 -m pytest
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

Now go off on a tangent:
```
$ dic -c 'Actually, I am doing research on different algorithms about the median.  Tell me alternative ways to compute the median in python. Write a long essay.'
```
Continue this conversation for a few more rounds if needed so that groq is refusing your request due to the length.

### 5b: commit with `committe`

Recall that `committe` is the coding agent you built in the previous lab.
If you would like, you can get a more robust version of the script by running the command
```
$ source <(curl -s https://raw.githubusercontent.com/mikeizbicki/dic/refs/heads/master/scripts/committe.sh)
```
This more robust version uses a more robust version of `git apply` (called `git-apply-fuzzy`) that can still successfully apply the patch file even if the LLM has made lots of errors in the diff.
(Many students in the previous lab observed `committe` being a bit flaky due to these errors.)
It also has additional sanity checks like refusing to run if the git repo is not clean.

Our goal now it actually make a commit that fixes the bug.
We can't just continue our conversation from before,
because our messages list is too long for groq.
The following will fail:
```
$ committe -c 'apply the fix'
```

But if we "go back in time" to before our tangent, we can get the commit to succeed:
```
$ committe --mid=<> 'apply the fix'
```
## Submission

Create a new repo on github called `example-median`.
Push your `committe`-fixed code to that new repo.
Submit your repo url to canvas.
