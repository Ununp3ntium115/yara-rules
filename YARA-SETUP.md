# Velociraptor Claw Edition - YARA Rules Setup Guide

This guide explains how to set up automated YARA rules updates using crontab or GitHub Actions.

## YARA Rules Sources (6 Total)

| Source | Description | Rules |
|--------|-------------|-------|
| **YARA Forge** | yarahq.github.io - Primary community source | ~11,000+ |
| **Citizen Lab** | Malware signatures from Citizen Lab | Variable |
| **macOS-Specific** | macOS malware rules (pedramamini) | ~50+ |
| **Awesome-YARA** | Curated rules from InQuest, Neo23x0/signature-base, Yara-Rules/rules | ~2,000+ |
| **YARAify/abuse.ch** | YARAhub community TLP:WHITE rules | Variable |
| **DetectRaptor** | mgreen27's Velociraptor YARA & VQL artifacts | Variable |

## Automation Options

### Option 1: GitHub Actions (Recommended)

GitHub Actions automatically updates rules daily at 3 AM UTC. No setup required - just push to the repository.

**Manual trigger:**
1. Go to GitHub repository
2. Click **Actions** tab
3. Select **Daily YARA Rules Update**
4. Click **Run workflow**

### Option 2: Local Crontab (macOS/Linux)

Set up automated daily updates on your local machine:

```bash
# Open crontab editor
crontab -e

# Add this line to run daily at 3 AM local time:
0 3 * * * /Users/brodynielsen/GitRepos/Velociraptor_ClawEdition/scripts/update-yara-rules.sh >> /var/log/yara-update.log 2>&1
```

**Verify crontab is set:**
```bash
crontab -l
```

### Option 3: launchd (macOS Recommended)

Create a launch agent for more reliable scheduling on macOS:

```bash
# Create the plist file
cat > ~/Library/LaunchAgents/com.velociraptorclaw.yara-update.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.velociraptorclaw.yara-update</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/brodynielsen/GitRepos/Velociraptor_ClawEdition/scripts/update-yara-rules.sh</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>3</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>
    <key>StandardOutPath</key>
    <string>/tmp/yara-update.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/yara-update-error.log</string>
    <key>RunAtLoad</key>
    <false/>
</dict>
</plist>
EOF

# Load the launch agent
launchctl load ~/Library/LaunchAgents/com.velociraptorclaw.yara-update.plist

# Verify it's loaded
launchctl list | grep velociraptorclaw
```

**To unload:**
```bash
launchctl unload ~/Library/LaunchAgents/com.velociraptorclaw.yara-update.plist
```

## Manual Update

Run the update script manually:

```bash
# Run the update script
./scripts/update-yara-rules.sh

# Or with full path
/Users/brodynielsen/GitRepos/Velociraptor_ClawEdition/scripts/update-yara-rules.sh
```

## Output Files

After running, you'll find:

| File | Description |
|------|-------------|
| `yara-rules/packages/full/combined-rules-master.yar` | Single combined rules file |
| `yara-rules/offline-package/velociraptor-claw-yara-offline.zip` | Comprehensive offline package |
| `yara-rules/version.json` | Version and source metadata |
| `yara-rules/vql-artifacts/` | Velociraptor VQL detection artifacts |
| `yara-rules/detection-data/` | CSV detection data files |
| `yara-rules/update.log` | Update script log |

## Using the Offline Package

The offline package (`velociraptor-claw-yara-offline.zip`) is designed for:

1. **Air-gapped environments** - Copy to isolated networks
2. **3rd party DFIR tools** - Import rules into other tools
3. **Velociraptor agents** - Distribute to remote endpoints

**Extract and use:**
```bash
unzip velociraptor-claw-yara-offline.zip -d /path/to/yara-rules/

# Scan with YARA CLI
yara -r /path/to/yara-rules/yara-rules/combined-rules-master.yar /path/to/scan

# Import VQL artifacts into Velociraptor
velociraptor artifacts install /path/to/vql-artifacts/*.yaml
```

## Integration with 3rd Party Tools

### YARA CLI
```bash
yara -s -r yara-rules/packages/full/combined-rules-master.yar /path/to/scan
```

### Volatility 3
```bash
vol -f memory.raw yarascan.YaraScan --yara-file yara-rules/packages/full/combined-rules-master.yar
```

### OSQuery
```sql
SELECT * FROM yara WHERE path LIKE '/suspicious/path/%'
  AND sigfile = '/path/to/yara-rules/packages/full/combined-rules-master.yar';
```

### Velociraptor
```vql
-- Use the built-in YARA scanning artifact
SELECT * FROM Artifact.Generic.Detection.Yara.Glob(
    PathGlob='C:/Windows/Temp/**',
    YaraRule='<paste rules or use file path>'
)
```

## DetectRaptor Submodule

DetectRaptor is included as a git submodule in the repository root:

```bash
# Initialize and update the submodule
git submodule init
git submodule update

# Update to latest
git submodule update --remote --merge DetectRaptor
```

The update script automatically pulls from DetectRaptor when it detects the submodule.

## Troubleshooting

### Crontab not running
- Check cron daemon is running: `sudo launchctl list | grep cron`
- Verify script is executable: `chmod +x scripts/update-yara-rules.sh`
- Check logs: `cat /var/log/yara-update.log`

### Download failures
- Check internet connectivity
- Verify GitHub and abuse.ch are accessible
- Review `yara-rules/update.log` for errors

### Disk space issues
- The full package can be 50-100MB+
- Clean old backups: `rm -rf yara-rules/packages.backup.*`
