# cli-nudnik

A Go-based CLI task reminder system that displays notifications across all terminal instances on macOS using Unix domain sockets, cron scheduling, and YAML configuration.

## Features

- ⏰ **Cron-based scheduling** - Use standard cron expressions for flexible reminder timing
- 🔔 **Cross-terminal notifications** - Reminders appear in ALL terminal instances (iTerm, VS Code, etc.)
- 📝 **YAML & CLI configuration** - Configure via config file or command-line flags
- 🎨 **Priority-based colors** - Visual priority indicators (low=green, medium=yellow, high=red, urgent=blinking red)
- 💾 **JSON storage** - File-based storage with atomic writes and file locking
- 🔧 **Shell integration** - Automatic setup for zsh, bash, and fish shells
- 🖥️ **Background daemon** - Persistent process for continuous monitoring

## Installation

1. **Build from source:**

   ```bash
   git clone <repository>
   cd cli-nudnik
   go build -o cli-nudnik
   sudo mv cli-nudnik /usr/local/bin/
   ```

2. **Set up shell integration (recommended):**
   ```bash
   cli-nudnik install
   ```

## Quick Start

1. **Start the daemon:**

   ```bash
   cli-nudnik daemon
   ```

2. **Add reminders:**

   ```bash
   # Daily standup at 9 AM weekdays
   cli-nudnik add "Daily standup" "0 9 * * 1-5"

   # Break reminder every 2 hours
   cli-nudnik add "Take a break" "0 */2 * * *" --priority medium

   # Urgent deadline reminder
   cli-nudnik add "Project deadline" "0 17 15 * *" --priority urgent --description "Final project submission due!"
   ```

3. **Connect terminals for notifications:**
   ```bash
   cli-nudnik monitor
   ```

## Commands

### Core Commands

- `cli-nudnik add [title] [cron-expression]` - Add a new reminder
- `cli-nudnik list` - List all reminders
- `cli-nudnik remove [id]` - Remove a reminder
- `cli-nudnik daemon` - Start the background daemon
- `cli-nudnik monitor` - Connect terminal to receive notifications
- `cli-nudnik install` - Set up shell integration

### Examples

```bash
# Add reminders with different schedules
cli-nudnik add "Morning coffee" "0 8 * * *"                    # Daily at 8 AM
cli-nudnik add "Weekly review" "0 17 * * 5"                    # Fridays at 5 PM
cli-nudnik add "Hourly stretch" "0 * * * *"                    # Every hour
cli-nudnik add "Monthly backup" "@monthly"                     # First of each month

# Add with options
cli-nudnik add "Important meeting" "30 14 * * 2" \
  --description "Team sync with stakeholders" \
  --priority high \
  --tags "meeting,work"

# Filter listings
cli-nudnik list --enabled                                      # Only enabled reminders
cli-nudnik list --priority urgent                             # Only urgent reminders
cli-nudnik list --tag work                                    # Filter by tag
```

## Cron Expression Format

```
# ┌───────────── minute (0 - 59)
# │ ┌───────────── hour (0 - 23)
# │ │ ┌───────────── day (1 - 31)
# │ │ │ ┌───────────── month (1 - 12)
# │ │ │ │ ┌───────────── day of week (0 - 6) (0 = Sunday)
# │ │ │ │ │
# * * * * *
```

**Special expressions:**

- `@yearly` or `@annually` - Run once a year
- `@monthly` - Run once a month
- `@weekly` - Run once a week
- `@daily` - Run once a day
- `@hourly` - Run once an hour

**Examples:**

- `0 9 * * 1-5` - Weekdays at 9 AM
- `*/15 * * * *` - Every 15 minutes
- `0 0 1 * *` - First day of every month at midnight
- `0 */6 * * *` - Every 6 hours

## Configuration

Default config file: `~/.cli-nudnik/config.yaml`

```yaml
daemon:
  socket_path: "~/.cli-nudnik/daemon.sock"
  log_file: "~/.cli-nudnik/daemon.log"
  pid_file: "~/.cli-nudnik/daemon.pid"

notifications:
  display_duration: "5s"
  urgency_colors:
    low: "green"
    medium: "yellow"
    high: "red"
    urgent: "red"
  show_timestamp: true
  bell: false

reminders:
  default_snooze: "5m"
  max_retries: 3

storage:
  data_file: "~/.cli-nudnik/reminders.json"
```

## Architecture

### Components

1. **CLI Commands** (`cmd/`) - Cobra-based command interface
2. **Data Models** (`internal/models/`) - Reminder and notification structures
3. **Storage Layer** (`pkg/storage/`) - JSON file storage with file locking
4. **Scheduler** (`pkg/scheduler/`) - Cron-based reminder scheduling
5. **Notifier** (`pkg/notifier/`) - Unix socket-based cross-terminal messaging
6. **Configuration** (`pkg/config/`) - YAML configuration management

### Communication Flow

```
┌─────────────┐    ┌──────────────┐    ┌─────────────────┐
│  Terminal 1 │    │   Terminal 2 │    │   Terminal N    │
│  (monitor)  │    │  (monitor)   │    │   (monitor)     │
└──────┬──────┘    └───────┬──────┘    └────────┬────────┘
       │                   │                    │
       └───────────────────┼────────────────────┘
                           │
            ┌──────────────▼──────────────┐
            │       Unix Socket           │
            │   (~/.cli-nudnik/daemon.sock)│
            └──────────────┬──────────────┘
                           │
            ┌──────────────▼──────────────┐
            │     Daemon Process          │
            │   - Scheduler (cron)        │
            │   - Notifier (broadcast)    │
            │   - Storage (JSON)          │
            └─────────────────────────────┘
```

### Terminal Integration

The shell integration automatically starts a monitor process in each new terminal:

**Zsh/Bash:**

```bash
# Added to ~/.zshrc or ~/.bashrc
if command -v cli-nudnik &> /dev/null; then
    cli-nudnik monitor --background &>/dev/null &
    disown
fi
```

**Fish:**

```fish
# Added to ~/.config/fish/config.fish
if command -sq cli-nudnik
    cli-nudnik monitor --background &>/dev/null &
    disown
end
```

## Development

### Build

```bash
go build -o cli-nudnik
```

### Test

```bash
go test ./...
```

### Dependencies

- `github.com/spf13/cobra` - CLI framework
- `github.com/spf13/viper` - Configuration management
- `github.com/robfig/cron/v3` - Cron scheduling
- `github.com/google/uuid` - UUID generation
- `github.com/fatih/color` - Terminal colors

## Troubleshooting

### Daemon won't start

- Check if socket path is accessible: `ls -la ~/.cli-nudnik/`
- Verify no other daemon is running: `ps aux | grep cli-nudnik`
- Check daemon logs: `tail -f ~/.cli-nudnik/daemon.log`

### Notifications not appearing

- Ensure daemon is running: `cli-nudnik daemon`
- Test connection: `cli-nudnik monitor --verbose`
- Check if monitor is running: `ps aux | grep "cli-nudnik monitor"`

### Permission errors

- Verify socket permissions: `ls -la ~/.cli-nudnik/daemon.sock`
- Try removing and recreating: `rm ~/.cli-nudnik/daemon.sock`

### Common Issues

1. **"Failed to connect to daemon"** - Start daemon first
2. **"Reminder not found"** - Use partial ID match: `cli-nudnik remove abc123`
3. **"Invalid cron expression"** - Test with online cron validators
4. **Shell integration not working** - Restart terminal after `cli-nudnik install`

## License

MIT License
