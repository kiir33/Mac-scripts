# iTerm2 Dev Session Helpers

This guide explains how to set up reusable shell commands that open and close a multi-pane iTerm2 development layout for the directory you are currently in.

The setup gives you three commands:

- `start_dev_session`
- `stop_dev_session`
- `restart_dev_session`

These commands are designed to:

- open a new iTerm2 window
- split it into panes
- run your development commands
- track the created window by the current working directory
- stop and close the correct window later from that same directory

## What This Setup Does

`start_dev_session` creates a new iTerm2 window and starts a predefined layout.

Example layout:

- top-left: Rails server
- top-right: frontend watcher / webpack / shakapacker
- bottom-left: Sidekiq
- bottom-right: clean shell

`stop_dev_session` looks up the window created for the current directory, sends `Ctrl+C`, then `Ctrl+D` to each pane, and finally closes that iTerm2 window.

`restart_dev_session` stops the tracked window for the current directory if it exists, then starts a fresh one.

## Requirements

- macOS
- iTerm2
- `zsh` or `bash`
- `osascript` available on the system

## Step 1: Create a Helper File

Create a file in your home directory:

```bash
touch ~/.dev-session
```

Add your helper functions to that file.

Use this template and replace the commands with whatever your project needs:

```bash
start_dev_session() {
  PROJECT_DIR="$PWD"
  STATE_DIR="$HOME/.dev-session-state"
  SESSION_KEY="$(printf '%s' "$PROJECT_DIR" | shasum -a 256 | awk '{print $1}')"

  mkdir -p "$STATE_DIR"

  WINDOW_ID="$(osascript <<EOF
tell application "iTerm2"
    set newWindow to (create window with default profile)
    set sessionTopLeft to current session of newWindow

    tell sessionTopLeft
        write text "cd \"$PROJECT_DIR\" && bundle exec rails server"
        set sessionTopRight to (split vertically with default profile)
    end tell

    tell sessionTopRight
        write text "cd \"$PROJECT_DIR\" && bin/shakapacker-dev-server"
        set sessionBottomRight to (split horizontally with default profile)
    end tell

    tell sessionBottomRight
        write text "cd \"$PROJECT_DIR\""
        write text "clear"
    end tell

    tell sessionTopLeft
        set sessionBottomLeft to (split horizontally with default profile)
    end tell

    tell sessionBottomLeft
        write text "cd \"$PROJECT_DIR\" && bundle exec sidekiq"
    end tell

    return id of newWindow
end tell
EOF
)"

  printf '%s\n' "$WINDOW_ID" > "$STATE_DIR/$SESSION_KEY.window"
  echo "Development session started for $PROJECT_DIR."
}

stop_dev_session() {
  PROJECT_DIR="$PWD"
  STATE_DIR="$HOME/.dev-session-state"
  SESSION_KEY="$(printf '%s' "$PROJECT_DIR" | shasum -a 256 | awk '{print $1}')"
  STATE_FILE="$STATE_DIR/$SESSION_KEY.window"

  if [ ! -f "$STATE_FILE" ]; then
    echo "No tracked dev session found for $PROJECT_DIR."
    return 1
  fi

  WINDOW_ID="$(tr -d '\n' < "$STATE_FILE")"

  osascript - "$WINDOW_ID" <<'APPLESCRIPT'
on run argv
    set targetWindowId to (item 1 of argv) as integer

    tell application "iTerm2"
        try
            set targetWindow to first window whose id is targetWindowId
        on error
            return "missing-window"
        end try

        set sessionList to {}

        repeat with aTab in tabs of targetWindow
            repeat with sess in sessions of aTab
                copy sess to end of sessionList
            end repeat
        end repeat

        repeat with sess in sessionList
            try
                tell sess
                    write text (ASCII character 3)
                end tell
            end try
        end repeat

        delay 0.3

        repeat with i from (count of sessionList) to 1 by -1
            try
                tell item i of sessionList
                    write text (ASCII character 4)
                end tell
            end try
        end repeat

        delay 0.5

        try
            close targetWindow
        end try
    end tell
end run
APPLESCRIPT

  rm -f "$STATE_FILE"
}

restart_dev_session() {
  stop_dev_session || true
  start_dev_session
}
```

## Step 2: Load the File From Your Shell Config

For `zsh`, add this to `~/.zshrc`:

```bash
if [ -f "$HOME/.dev-session" ]; then
  source "$HOME/.dev-session"
fi
```

For `bash`, add the same block to `~/.bashrc` or `~/.bash_profile`.

## Step 3: Reload Your Shell

For `zsh`:

```bash
source ~/.zshrc
```

For `bash`:

```bash
source ~/.bashrc
```

## Step 4: Use the Commands

Change into any project directory and run:

```bash
start_dev_session
```

Later, from that same directory:

```bash
stop_dev_session
```

To restart everything:

```bash
restart_dev_session
```

## Customizing the Commands

You can replace the example commands with anything you want, for example:

- `bundle exec rails s`
- `bundle exec sidekiq`
- `npm run dev`
- `pnpm dev`
- `bin/webpack-dev-server`
- `bin/shakapacker-dev-server`
- `pytest -k some_suite`

Important details:

- keep `PROJECT_DIR="$PWD"` if you want the functions to work from the current folder
- keep quotes around `"$PROJECT_DIR"` so paths with spaces still work
- keep the tracked `WINDOW_ID` logic if you want stop/restart to affect the right iTerm2 window

## Troubleshooting

If you get AppleScript syntax errors:

- make sure the `osascript` heredocs match exactly
- use `<<'APPLESCRIPT'` when you do not want shell interpolation inside the AppleScript block
- use plain `<<EOF` only when you intentionally want shell variables like `$PROJECT_DIR` expanded before AppleScript runs

If `stop_dev_session` says no tracked session was found:

- make sure you are in the same directory where you originally ran `start_dev_session`
- check whether the state file exists in `~/.dev-session-state/`

If iTerm2 does not respond:

- confirm the script uses `tell application "iTerm2"`
- grant Automation permissions if macOS prompts you
- test a small AppleScript manually with `osascript`

If pane closing fails intermittently:

- that usually means panes are disappearing while AppleScript is iterating them
- snapshotting sessions first, then closing them in reverse order, avoids most of those issues

## Notes

- This setup is intentionally shell-based, so teammates can copy it without modifying project code.
- The commands are general-purpose and can be reused across different repositories.
- The layout can be changed later without changing how the commands are invoked.
