# smart-codex

A hands-on guide to working in a large repository when it carries its own intelligence, built on a copy of [OpenAI Codex](https://github.com/openai/codex). Companion to [smart-ripgrep](https://github.com/gitsense/smart-ripgrep).

## What this repository is

This is a learning repository, not a project. The code is a copy of [OpenAI Codex](https://github.com/openai/codex), kept here as the dataset for the guide below. It is upstream's. The only things added are this README and the `.gitsense/` directory, which holds the Brains, their manifests, and the lessons this guide uses.

It is not affiliated with or endorsed by OpenAI. To use the real project, contribute, or file an issue, go to [openai/codex](https://github.com/openai/codex). Nothing here stands in for it. This copy exists only to show what a large codebase feels like to work in when it remembers things.

## The guide

The repo is the data, so every command below runs against the files in front of you, and you will see the same results, give or take small drift as the repo and the Brains change. We follow one real task the whole way:

> I want to add a new slash command.

An agent can do this without any of GitSense. ripgrep is fast and will search the whole tree in under a second. The question was never whether it can get the job done. It is how much time it spends getting there, and how much of that it spends again the next session.

A note on the numbers below. They were true the day this was written. The repo moves, so if you run a command and get a slightly different count, that is expected. The text rounds. The pasted output is exactly what the command printed.

## Setup

```bash
# Install gsc (read the script before you run it)
curl https://raw.githubusercontent.com/gitsense/chat/refs/heads/main/install.sh | bash

# Clone this repository
git clone https://github.com/gitsense/smart-codex
cd smart-codex

# Build the Brains from the committed manifests
gsc manifest import code-intent
gsc manifest import codex-rust-navigation
gsc manifest import rust-blast-radius

# Build the lessons Brain from the committed records
gsc lessons build
```

Then, inside your agent (Claude Code, Codex, and so on), run:

```
gsc experts init
```

That loads the Brain context, so the agent knows which Brains this repo has and can pick the right one when you ask in plain words. From here on, you talk to the agent, and it reaches for `gsc`.

Before trusting any of it, see how much of the repo a Brain actually knows about:

```
gsc coverage --db code-intent
```

About 95% of the repo is analyzed, 4,595 of 4,848 files. The rest is mostly binaries and files with no extension. Every Rust, TypeScript, Python, JSON, and Markdown file is in.

## Why this helps

A Brain is built to cut two things. The number of turns the agent takes, and the amount it has to hold in its head while it works. Fewer turns and less to carry means the agent moves faster, and so do you.

A coding agent with ripgrep can find what it needs. That is not the problem. The problem is the guessing about when. Is this the file I want for this change, or not? With nothing to go on, the only way to settle that is to open the file and read it. That costs a turn, and it fills up context. Do it across a dozen maybes and you have spent a lot of both before you change a line.

A Brain gives the agent enough to answer "is this the one" before it opens anything. That turns the work into fail fast. A dead end looks like a dead end right away, instead of after you have opened the file and read your way to the bottom. Seeing the dead end was always the cheap part. The Brain just lets you see it sooner.

With a Brain, the first pass usually gives one or two strong candidates with the reason each one matters, and a note on the ones that do not. So the next turn is confirming the right direction or seeing the dead end, not starting another search.

One more thing to keep straight as you read. ripgrep indexes the words. A Brain indexes the domain knowledge, what each file is for and when it matters.

## Get a count before you open anything

You want to add a slash command, so you go looking for where they live. You have seen big repos before, so you know `slash` will land in a lot of places. You tell the agent not to dump the matches:

> Find where slash commands live. It is going to be noisy, so just give me a file count first.

It runs:

```
rg -l slash | wc -l
```

```
144
```

About 150 files. That number earns its keep. It says this is noisy and you should not start opening things at random. But it is also all the number can tell you. In that list, a file that matters and a file that does not look exactly the same. The only way to tell them apart is to open them, and now you are back to a turn per file.

So you ask for the same overview with meaning attached:

> Now use the navigation Brain. Same search, but tell me what each file is for and when to skip it.

It runs:

```
gsc rg slash --db codex-rust-navigation --summary --fields purpose,skip_when
```

and tells you this:

```
#  Summary:  626 matches in 99 files

✓ codex-rs/tui/src/chatwidget/slash_dispatch.rs
; purpose: Modify this file to change slash command behavior, argument parsing, and command routing.
; skip_when: ["Changing core agent reasoning or tool execution logic that does not involve slash commands.", ...]

✓ codex-rs/tui/src/bottom_pane/chat_composer.rs
; purpose: Modify this file to change the TUI chat input behavior, including key routing, history navigation, slash command parsing, and large paste placeholder expansion.
; skip_when: ["Changing the actual execution logic of a slash command", ...]

✓ codex-rs/protocol/src/permissions.rs
; purpose: Modify this file to change how filesystem and network permissions are evaluated, including path resolution logic, glob matching behavior, and metadata protection rules.
; skip_when: ["Modifying TUI rendering or keybindings", ...]
```

First, notice it reports 99 files, not 144. The navigation Brain only covers Rust, so it already dropped the 45 matches in docs, snapshots, and the like. For a code change, that is mostly noise you are glad to lose.

Then read the notes. `permissions.rs` matched on path slashes and has nothing to do with slash commands. Its own note says it is about filesystem permissions. That is a dead end, and the agent can drop it without opening it. `slash_dispatch.rs` says it is where slash command behavior and routing live. That is the direction you want. `chat_composer.rs` is the interesting one. It does touch slash commands, it parses them from input, but its skip note says straight out that this is not where a command's behavior is added. A near miss, marked as a near miss, before anyone spent a turn finding that out.

Three calls made from one overview. The plain count could only tell you the pile was big.

## You don't know if it's simple until you're in it

Plain ripgrep would have gotten you there. Sometimes a job really is simple, and two or three greps and a couple of file reads are all it takes. The trouble is you cannot tell that at the start. "Add a slash command" might touch two files or twenty, and with 144 names the only way to find out is to start opening them. By the time you know it was simple, you have already paid to find out.

So you ask for the shape of the job first:

> Of those, which are actually about adding a slash command?

It runs:

```
gsc rg slash --db codex-rust-navigation --filter "agent_task_intents=add-slash-command" --summary --fields purpose
```

and tells you it is down to 13 files, all real slash command territory: the dispatch, the popup, the argument parsing, the filter for which commands show. 144 down to 13, and not a file was opened. It is a compass reading. You now know roughly where this lives, the TUI, about how big it is, a dozen files and not a hundred, and which ones are the define and dispatch core. The two files you will actually edit are in those 13, and the purpose lines point right at them.

On a short job, a compass like this saves you a little. On a long one it can save you the thing that matters most. A long task fills up context, and a model reasons worse the more it carries. Every file the agent did not open because the Brain called it a dead end is context kept clean for the real work.

## What you can do once there is a Brain

The last two sections were the same command, `gsc rg`, with different flags. That is most of the toolkit. Here is the rest, and when you reach for each.

- **See what each match is for.** Add `--fields purpose` and every file comes back with a line saying what it does. The everyday move. The difference between a list of names and a list you can judge.
- **Get an overview instead of a wall of code.** Add `--summary` and you get files, counts, and their notes, not every matching line. Use it when you want the shape before you read any of it.
- **Decide how much comes back.** `--fields` pulls only the columns you name. `--no-fields` pulls none. On a big result, that is the difference between a few thousand tokens and a few hundred thousand. You choose how much context to spend.
- **Narrow by meaning, not more text.** `--filter` cuts results down by what the files are, not by another search word. The compass move from the last section.
- **Drop the quiet matches.** `--min-matches N` hides files where the word shows up once or twice in passing. A cheap first cut when a word is common.
- **Stop searching for a word at all.** When you already know the intent, `gsc query` skips ripgrep and asks the Brain straight. It also turns up the files that never contain your word.
- **Ask a different Brain.** Risk and blast radius live in one Brain. Hard won lessons live in another. Same repo, different questions.

You do not need all of these at once. Most of the time it is `gsc rg <word> --db <brain> --fields purpose`, and you add a flag only when the results are still too big or too noisy to judge.

## Before you change it, see what leans on it

You have found the file where commands are defined, `slash_command.rs`. Before you touch it, the fair question is what else leans on it. Change a shared file and you can break something three directories away without ever seeing it.

You do not run anything for this. You ask:

> Before I change slash_command.rs, what depends on it, and is it risky?

It reaches for the blast radius Brain:

```
gsc query --db rust-blast-radius --glob "codex-rs/tui/src/slash_command.rs" --fields role,coupling_risk,exports_to
```

and tells you this. The file is marked `defines-api` and `coupling_risk: high`, and five files lean on it:

```
chat_composer.rs   the TUI chat input, key routing and slash parsing
command_popup.rs   the slash command popup, matching and visibility
slash_commands.rs  the filter for which commands are shown
slash_input.rs     how slash commands are parsed and completed
chatwidget.rs      the main chat surface
```

Now "affects five files" means something. Your change to the command list reaches the input parser, the popup, the availability filter, and the main chat surface. That is enough to judge whether the edit is safe, and nobody opened a file to find it out.

Here is what makes this different from running `syn` yourself. The edges are deterministic. `syn` read the code and knows exactly which file imports which, no guessing. The purpose next to each one is the AI part, the same domain knowledge the other Brains hold. The Brain is the two of them joined. The agent gets the exact graph and tells it to you in plain words.

And it does not stop at purpose. The Brain holds whatever fields you decide to put in it. Picture this same Brain built with ownership. Now when you ask what leans on a file, the answer also tells you who owns each piece, and the agent can tell you who to talk to before you touch a shared file, instead of finding that out in review.

## Cheap looks let you change your mind early

Everything so far cost almost nothing. A count, a summary, a filtered list, a blast radius check. No file was opened, so the context you are carrying is still small. That is worth more than it looks, because a cheap first look can change the job before you commit to it.

Look at what the overview told you. Not just "here are two files." The files that came back were a definition, a dispatch, a popup, a parser for arguments, and a filter for which commands are shown. That is the shape of the feature, and it raises questions your first sentence never answered. Does your command take an argument? Should it show in the popup, or stay hidden? Can someone run it while a task is going?

You ask:

> Looking at these, what choices does adding a command actually involve?

and from the same overview it already has, plus one open of the definition file to confirm, it tells you a command can take inline arguments, can be hidden from the popup, and can be allowed or blocked while a task runs. So you sharpen the request before writing anything:

> Add a /foo command that takes one argument and can run during a task.

Add it up. To get to the same place with `rg`, you open files to learn what they are, because the lines alone do not say. The careful version still opens half a dozen just to get oriented, and more to rule the near misses in or out. And opening is rarely one step. `chat_composer.rs` is eleven thousand lines. You do not read that in one pass. You page through it, or you search inside it and read around the hits, and every one of those is another round trip to take and think about. You did all of this here without opening a file, and you never spent that time on the ones that did not matter.

## Write down what you learned, so nobody learns it twice

Look back at what this one job taught you. Adding a slash command is two files, not one, the enum and the dispatch. The compiler will make you fill in the description and the during-a-task rule, but it will not point you at the dispatch file, so it is easy to add the command and have it do nothing. The composer and the tests look related and are not where commands get added. And the real choices turned out to be arguments, popup visibility, and whether it runs while a task is going.

That is a real piece of knowledge, and right now it lives in one place, this session. Close the terminal and it is gone. The next person to add a command, you next month, a teammate, the new hire in their first week, starts over from the noisy search.

A lesson is how you keep it. [smart-ripgrep](https://github.com/gitsense/smart-ripgrep) walks through what a lesson is and how to write one, so we will not repeat that here. You tell your agent:

> Save what we learned about adding a slash command, so the next session does not have to find it again.

It writes the lesson and you commit it. From then on it lives in the repo, next to the code.

Now jump to a fresh session, days later, nothing in context. Someone asks the plain question:

> I want to add a slash command. What lessons should I know of?

The agent checks the lessons Brain:

```
gsc query --db gsc-lessons --filter "tags=slash-command" --fields summary,review_checks
```

and tells them, before a single search or file open:

```
slash_command.rs              define the command here
chatwidget/slash_dispatch.rs  handle it here, or it does nothing

watch for:
- add a dispatch_command arm, not just an enum variant. a variant alone compiles but does nothing
- the description and during-a-task matches are compiler-forced. the dispatch one is not
- do not add command logic in chat_composer.rs, command_popup.rs, or the tests
```

The whole walkthrough you just read, the noisy search, the filtering, the blast radius check, the questions about arguments and the popup, none of it has to happen again. It happened once, and the answer stayed.

That is the part that compounds. The analyzer Brains tell you where things are, and you build those once. Lessons tell you what to watch for, and you gather those a session at a time. Every command someone adds, every sharp edge they hit, can go back in. The repo stops being just the code. It becomes what the team learned writing it, and the next agent starts from there instead of starting over.

## Where to go next

This was one task. The moves are the same for any of them. Search with meaning attached, narrow by intent, check what leans on a file before you change it, and write down what you learned so the next session does not start cold.

- [smart-ripgrep](https://github.com/gitsense/smart-ripgrep) is the companion guide on a smaller repo, and the place to start if Brains and lessons are new to you.
- [gsc-cli](https://github.com/gitsense/gsc-cli) is the open source tool. It imports Brains, writes lessons, and runs `gsc rg`.

---

<p align="center"><strong>Codex CLI</strong> is a coding agent from OpenAI that runs locally on your computer.
<p align="center">
  <img src="https://github.com/openai/codex/blob/main/.github/codex-cli-splash.png" alt="Codex CLI splash" width="80%" />
</p>
</br>
If you want Codex in your code editor (VS Code, Cursor, Windsurf), <a href="https://developers.openai.com/codex/ide">install in your IDE.</a>
</br>If you want the desktop app experience, run <code>codex app</code> or visit <a href="https://chatgpt.com/codex?app-landing-page=true">the Codex App page</a>.
</br>If you are looking for the <em>cloud-based agent</em> from OpenAI, <strong>Codex Web</strong>, go to <a href="https://chatgpt.com/codex">chatgpt.com/codex</a>.</p>

---

## Quickstart

### Installing and running Codex CLI

Run the following on Mac or Linux to install Codex CLI:

```shell
curl -fsSL https://chatgpt.com/codex/install.sh | sh
```

Run the following on Windows to install Codex CLI:

```
powershell -ExecutionPolicy ByPass -c "irm https://chatgpt.com/codex/install.ps1 | iex"
```

Codex CLI can also be installed via the following package managers:

```shell
# Install using npm
npm install -g @openai/codex
```

```shell
# Install using Homebrew
brew install --cask codex
```

Then simply run `codex` to get started.

<details>
<summary>You can also go to the <a href="https://github.com/openai/codex/releases/latest">latest GitHub Release</a> and download the appropriate binary for your platform.</summary>

Each GitHub Release contains many executables, but in practice, you likely want one of these:

- macOS
  - Apple Silicon/arm64: `codex-aarch64-apple-darwin.tar.gz`
  - x86_64 (older Mac hardware): `codex-x86_64-apple-darwin.tar.gz`
- Linux
  - x86_64: `codex-x86_64-unknown-linux-musl.tar.gz`
  - arm64: `codex-aarch64-unknown-linux-musl.tar.gz`

Each archive contains a single entry with the platform baked into the name (e.g., `codex-x86_64-unknown-linux-musl`), so you likely want to rename it to `codex` after extracting it.

</details>

### Using Codex with your ChatGPT plan

Run `codex` and select **Sign in with ChatGPT**. We recommend signing into your ChatGPT account to use Codex as part of your Plus, Pro, Business, Edu, or Enterprise plan. [Learn more about what's included in your ChatGPT plan](https://help.openai.com/en/articles/11369540-codex-in-chatgpt).

You can also use Codex with an API key, but this requires [additional setup](https://developers.openai.com/codex/auth#sign-in-with-an-api-key).

## Docs

- [**Codex Documentation**](https://developers.openai.com/codex)
- [**Contributing**](./docs/contributing.md)
- [**Installing & building**](./docs/install.md)
- [**Open source fund**](./docs/open-source-fund.md)

This repository is licensed under the [Apache-2.0 License](LICENSE).
