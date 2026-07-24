# Question 5 – Vi Recovery Mechanisms and Strategy

## Overview

When a system crashes while editing a file in vi or vim, several recovery mechanisms can help retrieve the unsaved work. Each operates at a different layer of the editor and offers different levels of completeness and reliability.


## Recovery Mechanisms

**Swap Files**

Vi automatically creates a hidden swap file while editing. For a file named config.conf the swap file is typically named .config.conf.swp and is placed in the same directory. The swap file records every change periodically, so it contains a snapshot of the buffer close to the moment of the crash. On the next open, vi detects the swap file and offers to recover from it.

Command to recover: vim -r config.conf

After recovery, the swap file must be deleted manually to prevent vi from prompting again on every subsequent open.

Command: rm .config.conf.swp


**Undo History (undofile)**

When the undofile option is enabled in .vimrc (set undofile), vim writes undo history to a persistent file in a configured undodir directory. This history survives crashes and can be replayed after reopening the file, letting the user undo or redo changes incrementally to reach a desired state.

Command in .vimrc:

set undofile
set undodir=~/.vim/undo


**Registers**

Vi stores copied and deleted text in named registers during a session. Registers are held in memory, so they are lost on a hard crash. They are not a reliable recovery tool for system failures but can recover accidental in-session deletions before a crash occurs.


**Backup Files**

When set backup is configured, vi writes a copy of the file as it existed before the current edit session into filename~. This is a pre-edit snapshot, not a real-time buffer. It protects against the user overwriting good content with bad edits but does not capture changes made during the crashed session.

Command in .vimrc: set backup


**Autorecovery via Swap**

This is not a separate mechanism but a term for the same swap-file recovery described above. Vim performs it automatically on startup when it detects an existing swap file for the target filename.


## Comparison Table

| Mechanism     | Survives Crash | Real-Time Updates | Default On |
|---------------|---------------|------------------|------------|
| Swap file     | Yes           | Yes (every few seconds) | Yes |
| Undofile      | Yes           | Yes              | No         |
| Backup file   | Yes (pre-edit) | No              | No         |
| Registers     | No            | N/A              | Yes        |


## Proposed Recovery Strategy

The most reliable strategy is to combine swap file recovery with persistent undo history enabled proactively before the crash.

Step 1 – Immediately after the crash, reopen the file

vim config.conf

Vi will detect the swap file and display a prompt. Choose Recover (r) to load the buffered state.


Step 2 – Save the recovered content immediately

:w config.conf.recovered

Save to a new name first to avoid any risk of the recovery write failing and losing the buffer.


Step 3 – Delete the swap file

:q
rm .config.conf.swp

Removing the swap file prevents vi from prompting on every future open.


Step 4 – Review with undo history if undofile was enabled

Open the recovered file and use u (undo) and Ctrl-R (redo) to navigate to the exact state before the crash. Undo history provides granular control that swap recovery alone cannot.


Step 5 – Verify the result

diff config.conf.recovered config.conf

Use diff to confirm the recovered version matches expectations before replacing the original.


## Justification

The swap file is the primary recovery tool because it is active by default, records changes in near real time, and is designed specifically to handle crashes. Backup files only save the pre-edit state and miss all work done during the session. Registers exist only in memory and are gone after a crash. Undo history is the ideal complement to swap recovery when enabled, because it allows the user to navigate the recovered state precisely. Without undofile, the undo history starts fresh on every session. The combination of swap recovery for breadth and undo history for precision represents the most complete and reliable strategy. Enabling set undofile in .vimrc before any important editing session is therefore the single most valuable preparation a developer can make.
