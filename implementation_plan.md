# Implementation Plan for Issue 2504

## Issue Summary
When a user presses Ctrl+C or ESC during a menu selection prompt (like the "approve/deny" menu in approve mode), the entire Goose application terminates instead of just canceling the current menu operation and returning to the main prompt.

## Relevant Codebase Analysis

### Issue Status: PARTIALLY FIXED ⚠️

**Issue 2504 has been partially resolved by commit `e9c93834cc` (PR #2586), but additional cases still need to be addressed.**

### What Has Been Fixed ✅

#### Commit Analysis: e9c93834cc - "fix program crashes and allow cancelling tool calls"

**Fixed Case: Tool Confirmation Menu**
- **Location:** `crates/goose-cli/src/session/mod.rs` (lines ~708-745)
- **Prompt Text:** "Goose would like to call the above tool, do you allow?"
- **Changes Made:**
  1. **Permission Enum Enhancement:** Added `Permission::Cancel` variant to `crates/goose/src/permission/permission_confirmation.rs`
  2. **Error Handling:** Replaced direct error propagation with proper error matching:
     ```rust
     let permission_result = cliclack::select(prompt).interact();
     let permission = match permission_result {
         Ok(p) => p,
         Err(e) => {
             if e.kind() == std::io::ErrorKind::Interrupted {
                 Permission::Cancel  // Convert Ctrl+C/ESC to Cancel
             } else {
                 return Err(e.into());
             }
         }
     };
     ```
  3. **Graceful Cancellation:** Added explicit handling for `Permission::Cancel` that returns to main prompt while preserving session state

### Remaining Cases to Fix ❌

#### 1. Context Length Exceeded Menu
- **Location:** `crates/goose-cli/src/session/mod.rs` (line 752)
- **Prompt Text:** "The model's context length is maxed out. You will need to reduce the # msgs. Do you want to?"
- **Current Code:** 
  ```rust
  let selected = cliclack::select(prompt)
      .item("clear", "Clear Session", "...")
      .item("truncate", "Truncate Messages", "...")
      .item("summarize", "Summarize Session", "...")
      .interact()?;  // ← Still crashes on Ctrl+C/ESC
  ```
- **Required Fix:** Apply the e9c93834cc pattern:
  1. **Add Cancel Option:** Add `.item("cancel", "Cancel", "Cancel and return to chat")`
  2. **Error Handling:** Replace `.interact()?` with `.interact()` and add error matching:
     ```rust
     let selected_result = cliclack::select(prompt)
         .item("clear", "Clear Session", "...")
         .item("truncate", "Truncate Messages", "...")
         .item("summarize", "Summarize Session", "...")
         .item("cancel", "Cancel", "Cancel and return to chat")
         .interact();
     
     let selected = match selected_result {
         Ok(s) => s,
         Err(e) => {
             if e.kind() == std::io::ErrorKind::Interrupted {
                 "cancel"  // Convert Ctrl+C/ESC to Cancel
             } else {
                 return Err(e.into());
             }
         }
     };
     ```
  3. **Handle Cancellation:** Add `"cancel" => { break; }` to return to main prompt

#### 2. Summarization Confirmation Menu
- **Location:** `crates/goose-cli/src/session/mod.rs` (line 557)
- **Prompt Text:** "Are you sure you want to summarize this conversation? This will condense the message history."
- **Current Code:**
  ```rust
  let should_summarize = cliclack::confirm(prompt).initial_value(true).interact()?;  // ← Crashes on Ctrl+C/ESC
  ```
- **Required Fix:** Apply the e9c93834cc pattern:
  1. **Convert to Select:** Replace `cliclack::confirm` with `cliclack::select` to allow explicit Cancel option:
     ```rust
     let summarize_result = cliclack::select(prompt)
         .item(true, "Yes", "Summarize the conversation")
         .item(false, "No", "Keep the conversation as is")
         .item("cancel", "Cancel", "Cancel and return to chat")
         .interact();
     
     let should_summarize = match summarize_result {
         Ok(choice) => match choice {
             true => true,
             false => false,
             "cancel" => {
                 continue; // Return to main prompt
             }
             _ => unreachable!()
         },
         Err(e) => {
             if e.kind() == std::io::ErrorKind::Interrupted {
                 continue; // Convert Ctrl+C/ESC to Cancel, return to main prompt
             } else {
                 return Err(e.into());
             }
         }
     };
     ```
  2. **Alternative Approach:** Keep `cliclack::confirm` but handle interruption as "No":
     ```rust
     let should_summarize = match cliclack::confirm(prompt).initial_value(true).interact() {
         Ok(choice) => choice,
         Err(e) => {
             if e.kind() == std::io::ErrorKind::Interrupted {
                 false // Convert Ctrl+C/ESC to "No"
             } else {
                 return Err(e.into());
             }
         }
     };
     ```

#### 3. Plan Action Confirmation Menu
- **Location:** `crates/goose-cli/src/session/mod.rs` (line 627)
- **Prompt Text:** "Do you want to clear message history & act on this plan?"
- **Current Code:**
  ```rust
  let should_act = cliclack::confirm("Do you want to clear message history & act on this plan?")
      .initial_value(true)
      .interact()?;  // ← Crashes on Ctrl+C/ESC
  ```
- **Required Fix:** Apply the e9c93834cc pattern:
  1. **Convert to Select:** Replace `cliclack::confirm` with `cliclack::select` to allow explicit Cancel option:
     ```rust
     let action_result = cliclack::select("Do you want to clear message history & act on this plan?")
         .item(true, "Yes", "Clear history and act on the plan")
         .item(false, "No", "Keep the plan as a message")
         .item("cancel", "Cancel", "Cancel and return to chat")
         .interact();
     
     let should_act = match action_result {
         Ok(choice) => match choice {
             true => true,
             false => false,
             "cancel" => {
                 // Return to main prompt without adding plan response
                 return Ok(());
             }
             _ => unreachable!()
         },
         Err(e) => {
             if e.kind() == std::io::ErrorKind::Interrupted {
                 // Convert Ctrl+C/ESC to Cancel, return to main prompt
                 return Ok(());
             } else {
                 return Err(e.into());
             }
         }
     };
     ```
  2. **Alternative Approach:** Keep `cliclack::confirm` but handle interruption as "No":
     ```rust
     let should_act = match cliclack::confirm("Do you want to clear message history & act on this plan?")
         .initial_value(true)
         .interact() {
         Ok(choice) => choice,
         Err(e) => {
             if e.kind() == std::io::ErrorKind::Interrupted {
                 false // Convert Ctrl+C/ESC to "No"
             } else {
                 return Err(e.into());
             }
         }
     };
     ```

### Implementation Strategy

#### For `cliclack::select` Cases:
1. Replace `.interact()?` with `.interact()`
2. Add error matching to handle `std::io::ErrorKind::Interrupted`
3. Add "Cancel" option to menu items
4. Handle cancellation by returning to main prompt with preserved state

#### For `cliclack::confirm` Cases:
1. Replace `.interact()?` with `.interact()`
2. Add error matching to handle `std::io::ErrorKind::Interrupted`
3. Treat interruption as "No" response or explicit cancellation
4. Return to main prompt with preserved state

### Exclusions

**Other `cliclack` Usages:** The codebase contains additional `cliclack::select` and `cliclack::confirm` calls in:
- `crates/goose-cli/src/commands/configure.rs` (multiple instances)
- `crates/goose-cli/src/commands/project.rs` (multiple instances)
- `crates/goose-cli/src/session/builder.rs` (one instance)

These cases of menu selection are not in the context of an active goose session. Exclude them from the scope of this fix. 
