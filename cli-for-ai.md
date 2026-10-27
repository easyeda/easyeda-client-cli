# For AI: EasyEDA Pro Command-Line Client

This document is written for **AI coding assistants (agents)** such as Claude Code, OpenCode, QwenCode, or GitHub Copilot, and assumes the executor can run terminal commands and read/write the file system.

If you are a human developer, read the human-oriented [Command-Line Client](https://prodocs.easyeda.com/en/api/guide/cli.html) page instead.

> **The CLI and the extension API are still evolving.** Command names, options, API signatures, and design-file fields may change between client versions. This document therefore does **not** list commands, APIs, or examples. It only teaches how to discover them at runtime. Whenever help output differs from anything you remember or from any other document, **trust the help output of the installed version**.

## When to Use

Read this document when the user asks you to drive EasyEDA Pro from the command line, for example:

- draw a schematic or PCB programmatically;
- generate or modify an `.eprj3` project;
- export a netlist, BOM, Gerber, STEP, or screenshot;
- any other request that requires operating the EasyEDA Pro editor from a script.

## Hard Facts

| Fact | Description |
| --- | --- |
| Product | EasyEDA Pro **desktop client** (CLI requires v4.1.60 or above). The browser / online edition does not support sessions or the bridge |
| Executable | `easyeda-pro` (international) or `lceda-pro` (mainland China). If the command is not found, ask the user to install or locate the client; do not install it yourself |
| Bridge | Every command talks to the running editor through a local bridge |
| Session | One editor window equals one session. The session is created by a command and returns a session ID; later commands need that ID |
| Response | Every command returns the same JSON envelope. Check the top-level `ok` field first; on failure read the error message |
| Source of truth | The `--help` output of the installed executable, plus the built-in `doc` reference |

## Discover Commands

Always discover, never assume. Use the three levels of help in this order.

### 1. List all commands

```shell
easyeda-pro --help
```

The help header prints the exact product name and version — record it, since your later findings apply to that version. The command list is the authoritative set of available commands and subcommands.

If the executable is missing entirely, stop and ask the user to install the EasyEDA Pro desktop client.

### 2. Find a command by keyword

When you do not know the command name:

```shell
easyeda-pro --search <keyword>
```

Use English keywords (for example the domain you are working in). The result lists matching commands with one-line descriptions; pick the one that matches the user's intent and continue to the next step.

### 3. Read a command's usage

Every command and subcommand documents itself:

```shell
easyeda-pro <command> --help
easyeda-pro <command> <subcommand> --help
```

The help output gives the exact usage line, every option, defaults, accepted values, and any constraints. **Read it before running the command**; do not guess option names or shapes. If a command needs a value only the user can decide (project name, target path, and similar), ask the user instead of inventing one.

## Discover the Editor API and File Format

The CLI also serves the editor's reference documentation, so you never need to rely on memorized API knowledge:

- List the API classes exposed by the editor;
- Read the methods of one class, then the full signature of one method;
- Inspect enumerations;
- Read the design-file format field definitions.

All of this is reached through the `doc` command family. Start with `easyeda-pro doc --help`, then follow the subcommand help as described above; the available namespaces and options are printed there. Read the relevant signature or format type **before** writing code that uses it, and re-check it whenever the client version changes.

## Rules for Acting on the User's Behalf

1. **Discover from help, don't recall.** Every command, option, parameter, and API signature you use must be confirmed from `--help` or `doc` output of the installed version. Do not copy signatures from memory, older docs, or generated scripts without re-checking them.
2. **Check `ok` on every response.** On failure, read the error message and adapt; do not continue as if the step succeeded.
3. **Verify artifacts on disk** when a command is supposed to write files.
4. **Confirm values that require user decisions** — project name, target directory, what to build or export — before running commands. Never invent them.
5. **Use forward slashes in paths** (`C:/Users/...`). Backslashes are easily mangled through shells and JSON.
6. **Write only inside the project directories.** Ask the user or inspect the environment configuration for the allowed paths before writing.
7. **Announce long operations** to the user before running them.
8. **Close sessions you created** once they are no longer needed, using the session-closing command shown by the help, so editor windows do not pile up.
9. **GUI-only steps are the user's job.** If a step requires interaction with the client UI or a setting the CLI cannot change, stop and ask the user.

## What to Report When Done

Report concisely:

- the client version you worked against;
- the commands you actually ran (or the final help-derived workflow);
- the produced artifacts and their absolute paths;
- the session shutdown result, and any step that failed with its error message.
