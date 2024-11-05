# File Sync to S3 with macOS Notifications
![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)
![Platform: macOS](https://img.shields.io/badge/platform-macOS-blue)
![Shell: Bash](https://img.shields.io/badge/shell-bash-89E051)

A lightweight macOS solution that automatically monitors file changes and syncs them to AWS S3 in real-time. When a file is modified, the system instantly uploads it to your specified S3 bucket and sends a desktop notification confirming the successful sync. Perfect for backing up important files, sharing updates across devices, or maintaining cloud backups of critical documents.

## Prerequisites

- macOS
- Homebrew
- AWS account with configured profile
- Created S3 bucket in AWS account

## Installation

1. Install required tools:
```bash
brew install fswatch awscli
```

2. Configure AWS (if not already done):
```bash
aws configure --profile your_profile_name
```

3. Create directory structure:
```bash
mkdir -p ~/system_scripts/logs
```

4. Create the monitoring script:
```bash
touch ~/system_scripts/passwords_sync.sh
chmod 755 ~/system_scripts/passwords_sync.sh
```

5. Add this content to `passwords_sync.sh`:
```bash
#!/bin/bash

# Configuration
FILE_PATH="/path/to/file"            # Your file to monitor
S3_BUCKET="your-bucket-name"         # Your S3 bucket
S3_PATH="path/in/bucket"             # Path in your bucket
AWS_PROFILE_NAME="your-profile"      # Your AWS profile from ~/.aws/config
LOG_FILE="$HOME/system_scripts/logs/passwords_sync.log"

# Check requirements
if ! command -v fswatch >/dev/null 2>&1; then
    echo "Error: fswatch not installed"
    echo "Install with: brew install fswatch"
    exit 1
fi

if ! command -v aws >/dev/null 2>&1; then
    echo "Error: AWS CLI not installed"
    echo "Install with: brew install awscli"
    exit 1
fi

# Check if AWS profile exists
if ! aws configure list-profiles | grep -q "^${AWS_PROFILE_NAME}$"; then
    echo "Error: AWS Profile '$AWS_PROFILE_NAME' not found in ~/.aws/config"
    echo "Available profiles:"
    aws configure list-profiles
    exit 1
fi

# Export AWS Profile
export AWS_PROFILE="$AWS_PROFILE_NAME"

# Function to send macOS notification
notify() {
    osascript -e "display notification \"$1\" with title \"S3 Syncronization successfull\" sound name \"Glass\""
}

# Create log file if it doesn't exist
touch "$LOG_FILE"

echo "Starting file monitor..."
echo "Watching: $FILE_PATH"
echo "S3 destination: s3://$S3_BUCKET/$S3_PATH"
echo "AWS Profile: $AWS_PROFILE_NAME"
echo "Log file: $LOG_FILE"
echo "Use Ctrl+C to stop monitoring"

# Start monitoring
fswatch -0 "$FILE_PATH" | while read -d "" event; do 
    if aws s3 cp "$event" "s3://$S3_BUCKET/$S3_PATH" --profile "$AWS_PROFILE_NAME"; then
        timestamp=$(date '+%Y-%m-%d %H:%M:%S')
        message="File uploaded to s3://$S3_BUCKET/$S3_PATH"
        echo "$timestamp - $message" >> "$LOG_FILE"
        notify "$message"
    fi
done
```

6. Create LaunchAgent:
```bash
touch ~/Library/LaunchAgents/com.user.passwordssync.plist
```

7. Add this content to the plist file:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.user.passwordssync</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>/Users/YOUR_USERNAME/system_scripts/passwords_sync.sh</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/Users/YOUR_USERNAME/system_scripts/logs/passwords_sync.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/YOUR_USERNAME/system_scripts/logs/passwords_sync.log</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
    </dict>
</dict>
</plist>
```

## Example Configuration

Here's an example of how to configure the script:

```bash
FILE_PATH="/Users/username/Documents/myfile.txt"
S3_BUCKET="my-backup-bucket"
S3_PATH="backups/myfile.txt"
AWS_PROFILE_NAME="personal"
```

## Usage and Management

Start the service:
```bash
launchctl load ~/Library/LaunchAgents/com.user.passwordssync.plist
```

Check status:
```bash
# View service status
launchctl list | grep passwordssync

# View logs
tail -f ~/system_scripts/logs/passwords_sync.log
```

Stop service:
```bash
launchctl unload ~/Library/LaunchAgents/com.user.passwordssync.plist
```

## Troubleshooting

Common issues:
- If fswatch not found: Verify PATH in plist
- If AWS credentials not working: Check profile in `~/.aws/config`
- If notifications not showing: Check macOS notification settings

Test commands:
```bash
# Test AWS connection
aws s3 ls --profile your_profile_name

# Test notifications
osascript -e 'display notification "Test" with title "S3 Upload" sound name "Glass"'
```

## Uninstall

```bash
# Stop service
launchctl unload ~/Library/LaunchAgents/com.user.passwordssync.plist

# Remove files
rm ~/Library/LaunchAgents/com.user.passwordssync.plist
rm -rf ~/system_scripts
```