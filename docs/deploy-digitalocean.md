# Deploying Claude Code Telegram Bot on DigitalOcean

This guide walks through deploying [claude-code-telegram](https://github.com/RichardAtCT/claude-code-telegram) on a DigitalOcean Droplet so your bot runs 24/7.

## Requirements

- A [DigitalOcean](https://www.digitalocean.com/) account
- A Telegram bot token from [@BotFather](https://t.me/botfather)
- Your Telegram user ID (message [@userinfobot](https://t.me/userinfobot) to get it)
- Claude authentication (API key or CLI auth)

## 1. Create a Droplet

1. Log in to DigitalOcean and click **Create > Droplets**
2. Choose **Ubuntu 24.04 LTS**
3. Select a plan:
   - **Basic $6/mo (1 vCPU, 1 GB RAM)** -- works for light usage
   - **$12/mo (2 vCPU, 2 GB RAM)** -- recommended for production
4. Choose a datacenter region close to you
5. Add your SSH key (or set a root password)
6. Click **Create Droplet**

## 2. Initial Server Setup

SSH into your droplet:

```bash
ssh root@YOUR_DROPLET_IP
```

Create a non-root user and set up basics:

```bash
# Create user
adduser botuser
usermod -aG sudo botuser

# Set up firewall
ufw allow OpenSSH
ufw enable

# If using webhooks, open the API server port
# ufw allow 8080/tcp

# Switch to the new user
su - botuser
```

## 3. Install Dependencies

```bash
# System packages
sudo apt update && sudo apt install -y python3.11 python3.11-venv git curl tmux

# Install Poetry
curl -sSL https://install.python-poetry.org | python3 -

# Add Poetry to PATH
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Verify
poetry --version
python3.11 --version
```

### Install Claude Code CLI (if using CLI auth)

```bash
# Install Node.js (required for Claude CLI)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Install Claude Code CLI
npm install -g @anthropic-ai/claude-code

# Authenticate
claude auth login
```

## 4. Install the Bot

### Option A: From a release tag (recommended)

```bash
cd ~
pip install git+https://github.com/RichardAtCT/claude-code-telegram@latest
```

### Option B: From source

```bash
cd ~
git clone https://github.com/RichardAtCT/claude-code-telegram.git
cd claude-code-telegram
make dev
```

## 5. Configure

```bash
cd ~/claude-code-telegram   # if installed from source
cp .env.example .env
nano .env
```

Set these required values:

```bash
TELEGRAM_BOT_TOKEN=your-bot-token-here
TELEGRAM_BOT_USERNAME=your_bot_username
APPROVED_DIRECTORY=/home/botuser/projects
ALLOWED_USERS=your-telegram-user-id

# Authentication (pick one):
# Option A: CLI auth -- no key needed if you ran `claude auth login`
# Option B: API key
ANTHROPIC_API_KEY=sk-ant-api03-your-key-here

# Production settings
ENVIRONMENT=production
DEBUG=false
LOG_LEVEL=INFO
RATE_LIMIT_REQUESTS=5
CLAUDE_MAX_COST_PER_USER=5.0
SESSION_TIMEOUT_HOURS=12
```

Create the projects directory:

```bash
mkdir -p /home/botuser/projects
```

## 6. Run with systemd (Recommended)

Create a systemd service for automatic start on boot and restart on failure:

```bash
sudo tee /etc/systemd/system/claude-telegram.service > /dev/null << 'EOF'
[Unit]
Description=Claude Code Telegram Bot
After=network.target

[Service]
Type=simple
User=botuser
WorkingDirectory=/home/botuser/claude-code-telegram
EnvironmentFile=/home/botuser/claude-code-telegram/.env
ExecStart=/home/botuser/.local/bin/poetry run python -m src.main
Restart=always
RestartSec=10

# Security hardening
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/home/botuser/projects /home/botuser/claude-code-telegram

[Install]
WantedBy=multi-user.target
EOF
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable claude-telegram
sudo systemctl start claude-telegram
```

### Managing the service

```bash
sudo systemctl status claude-telegram    # Check status
sudo systemctl restart claude-telegram   # Restart
sudo systemctl stop claude-telegram      # Stop
journalctl -u claude-telegram -f         # View live logs
journalctl -u claude-telegram --since "1 hour ago"  # Recent logs
```

## 7. Alternative: Run with tmux

If you prefer a simpler setup without systemd:

```bash
tmux new-session -d -s claude-bot "cd ~/claude-code-telegram && make run"
```

Manage the session:

```bash
tmux attach -t claude-bot     # View output
# Press Ctrl+B, then D to detach
tmux kill-session -t claude-bot  # Stop
```

## 8. Enable Webhooks (Optional)

If you want GitHub webhooks or other external triggers:

Add to your `.env`:

```bash
ENABLE_API_SERVER=true
API_SERVER_PORT=8080
GITHUB_WEBHOOK_SECRET=your-generated-secret
NOTIFICATION_CHAT_IDS=your-telegram-user-id
```

### Set up Nginx as reverse proxy with SSL

```bash
sudo apt install -y nginx certbot python3-certbot-nginx

# Create Nginx config
sudo tee /etc/nginx/sites-available/claude-telegram << 'EOF'
server {
    listen 80;
    server_name your-domain.com;

    location /webhooks/ {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF

sudo ln -s /etc/nginx/sites-available/claude-telegram /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx

# Get SSL certificate
sudo certbot --nginx -d your-domain.com
```

Then configure your GitHub webhook URL as `https://your-domain.com/webhooks/github`.

## 9. Set Up Backups (Optional)

The bot stores data in SQLite. Back it up daily:

```bash
# Create backup script
mkdir -p ~/backups
cat > ~/backup-bot.sh << 'SCRIPT'
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="$HOME/backups"
DB_PATH="$HOME/claude-code-telegram/data"

# Copy SQLite databases
cp "$DB_PATH"/*.db "$BACKUP_DIR/" 2>/dev/null
cp "$HOME/claude-code-telegram/.env" "$BACKUP_DIR/.env.backup"

# Keep only last 7 days
find "$BACKUP_DIR" -name "*.db" -mtime +7 -delete
SCRIPT
chmod +x ~/backup-bot.sh

# Schedule daily backup at 3 AM
(crontab -l 2>/dev/null; echo "0 3 * * * /home/botuser/backup-bot.sh") | crontab -
```

## 10. Monitoring

### Health check with uptime monitoring

You can use DigitalOcean's built-in monitoring, or set up a simple health check:

```bash
# Add a cron job to check if the bot is running
(crontab -l 2>/dev/null; echo "*/5 * * * * systemctl is-active claude-telegram || systemctl restart claude-telegram") | crontab -
```

### Resource monitoring

```bash
# Check memory usage
free -h

# Check disk usage
df -h

# Check bot process
ps aux | grep claude
```

## Updating the Bot

```bash
cd ~/claude-code-telegram
git pull origin main
make dev  # reinstall dependencies
sudo systemctl restart claude-telegram
```

Or if installed via pip:

```bash
pip install --upgrade git+https://github.com/RichardAtCT/claude-code-telegram@latest
sudo systemctl restart claude-telegram
```

## Cost Estimate

| Component | Cost |
|-----------|------|
| DigitalOcean Droplet (2GB) | ~$12/mo |
| Domain name (optional) | ~$10/yr |
| Claude API usage | Variable (use `CLAUDE_MAX_COST_PER_USER` to cap) |
| **Total** | **~$12-15/mo + API costs** |

## Troubleshooting

**Bot won't start:**
```bash
journalctl -u claude-telegram -n 50 --no-pager
```

**Permission errors:**
```bash
# Ensure botuser owns the project directory
sudo chown -R botuser:botuser /home/botuser/projects
```

**Out of memory:**
- Upgrade to a larger Droplet, or add swap:
```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

**Claude CLI auth issues over SSH:**
- Use `ANTHROPIC_API_KEY` instead of CLI auth to avoid keychain/credential store issues on headless servers.
