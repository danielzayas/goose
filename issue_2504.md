# Issue 2504

Summary of github issue 2504 from https://github.com/block/goose/issues/2504.

## Issue Description

Summary of the Issue: This bug, clearly detailed by user bendavis78 , affects the Command Line Interface (CLI) of "goose" (specifically tested on Arch Linux, Goose version 1.0.22). When the application presents a menu selection prompt (e.g., an "approve/deny" choice), pressing either the Ctrl+C key combination or the ESC key causes the entire application to terminate. The expected behavior, as explicitly outlined , is for these actions to cancel the current menu selection and return the user to the main ( O)> goose prompt, while preserving any previous command history, messages, and operational context.

In the Goose CLI, pressing Ctrl+C or Esc while a selection menu is open currently causes the entire application to quit. The expected behavior is that these keys should simply cancel the menu and return the user to the main Goose prompt, preserving any previous session context.

During a menu selection prompt such as the "approve/deny" menu, if I press Ctrl+C or ESC, it should end the menu and put the user back into a normal prompt cursor.

## GOOSE_MODE background

The GOOSE_MODE configuration allows the user to specific if/when they would like the goose application to ask for approval before performing actions that can mutate state (CLI command, tool use invovation, and I/O like file system changes):
1. "auto" mode configures goose perform all such actions without asking for user approval. 
2. "approve" mode configures goose to always as for user approval before performing such actions. 

The GOOSE_MODE is typically specified by a line in `~/.config/goose/config.yaml` such as:
```
GOOSE_MODE: auto
```
or
```
GOOSE_MODE: approve
```

## Steps to reproduce the issue

Steps to reproduce the behavior:

1. Set GOOSE_MODE to "approve" if not already set. 
2. Start a new goose session via command like `./target/debug/goose session`.
3. Submit any prompt that requires reading a file. E.g. "read the README file in the project root."
4. Wait until it prompts for the "approve/deny" selection.
5. Press Ctrl+C or ESC.

The program will exit completely (no error code returned).

## Desired behavior

Pressing Ctrl+C or ESC during a menu selection should return the user to the main goose prompt. E.g. The "( O)>" prompt in the CLI. Any previous history, messages, context should be preserved.

## Create an implementation plan

Your first goal is to write and edit the `implementation_plan.md`:
* Focus on the CLI application. The `./crates/goose-cli` should be very relevant. 
* Begin exploring the source code related to the CLI. Look for modules or functions that handle user input, display menus, and particularly any code that might be responsible for catching operating system signals like SIGINT or processing special keystrokes such as ESC.
* Add summaries of the relevant files to `implementation_plan.md`. Note all existing code for SIGNIT and ESC handling. 
* Proposed detailed techinical implementation steps without writiing any code. The core of the fix will likely involve modifying this input handling logic. The goal is to catch these events during menu selection and, instead of allowing the application to exit, transition the application state back to the main prompt gracefully. 
* Only modify the the `implementation_plan.md` file. Do not add or edit and other files. 
