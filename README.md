# openclaw Setup Guide for Ubuntu (on Oracle Cloud)

This guide outlines the steps to install, secure, and optimize **openclaw** on an (pre-built) ARM-based Ubuntu server.
(mine 4CPU, 24GB ram, 100GB HDD)

![status](https://github.com/inchinet/openclaw/blob/main/status.png) 

## 1. Prerequisites
Ensure you have Node.js 22+ installed.
```bash
# Install Node.js 22
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs
```

## 2. Install openclaw
Install the global package and verify:
```bash
sudo npm i -g openclaw
openclaw --version  # (Targeting 2026.2.6-3)

openclaw onboard    # to configure
```

## 3. Run as a Service (Systemd)
Create a service file to handle background execution and API key injection:
`sudo nano /etc/systemd/system/openclaw.service`

```ini
[Service]
Type=simple
User=ubuntu
EnvironmentFile=/etc/openclaw.env
# Injects secrets from .env into the bot's config on every start
ExecStartPre=/usr/bin/python3 -c "import os, json; p='/home/ubuntu/.openclaw/agents/main/agent/auth-profiles.json'; d=json.load(open(p)) if os.path.exists(p) else {'version':1,'profiles':{'google:default':{'type':'api_key','provider':'google'},'openrouter:default':{'type':'api_key','provider':'openrouter'}}}; d['profiles']['google:default']['key']=os.getenv('GOOGLE_API_KEY'); d['profiles']['openrouter:default']['key']=os.getenv('OPENROUTER_API_KEY'); json.dump(d, open(p, 'w'), indent=2)"
ExecStart=/usr/bin/openclaw gateway --port 18789
Restart=on-failure
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

## 4. Secrets & Security (Consolidated)
This section covers API keys, file permissions, and firewall safety.

### A. API Key Environment
Create `/etc/openclaw.env`:
```bash
GOOGLE_API_KEY=your_actual_key
OPENROUTER_API_KEY=your_actual_key
```

setup and enable your google API in https://console.cloud.google.com/apis/

### B. Lockdown Permissions (Critical)
Always use `chmod 600` for files containing keys:
```bash
# Root env file
sudo chown root:root /etc/openclaw.env
sudo chmod 600 /etc/openclaw.env

# Agent auth profile
chmod 600 ~/.openclaw/agents/main/agent/auth-profiles.json
```

### C. Fail2ban Integration
Assume fail2ban is installed in your server
Add to `/etc/fail2ban/jail.local`:
```ini
[openclaw]
enabled = true
port    = 18789
backend = systemd
journalmatch = _SYSTEMD_UNIT=openclaw.service
```

## 5. Web Access & Reverse Proxy
Proxy traffic through Apache to match `https://your-url`.

1. **Enable Modules**: `sudo a2enmod proxy proxy_http proxy_wstunnel`
2. **Apache Config**: Add to your SSL VirtualHost:
```apache
<VirtualHost *:443>
    ServerName your-url
    
    ProxyPreserveHost On
    ProxyPass / http://127.0.0.1:18789/
    ProxyPassReverse / http://127.0.0.1:18789/

    <Location />
        AuthType Basic
        AuthName "openclaw Private Access"
        AuthUserFile /etc/apache2/.htpasswd
        Require valid-user
    </Location>
</VirtualHost>
```

## 6. Models & ARM Optimization
Running local Models on ARM CPUs requires care. 
Ollama setup refer https://docs.openclaw.ai/providers/ollama
local LLM consideration: https://blog.darkthread.net/blog/openclaw-w-local-llm/


install Ollama
```bash
curl -fsSL https://ollama.com/install.sh | sh
```
download and run a model
```bash
ollama run qwen2:0.5b
ollama run gemma2:2b

ollama list   # to see models installed
```

### A. The "Sweet Spot" (Ollama)
Without a GPU, avoid models $> 3$B.
- **Recommended**: `qwen2:0.5b` (352MB) - Fast responses on CPU.
- **Max Weight**: `gemma2:2b` (1.6GB) - Slower but manageable.
- **Delete Heavy Models**: `ollama rm llama3.1:8b mistral:7b` (Prevents 400% CPU spikes).

### B. Gemini 2.5 Flash (Primary)
Use cloud models for instant WhatsApp responses.
In `~/.openclaw/openclaw.json`:
```json
"agents": {
  "defaults": {
    "model": {
      "primary": "google/gemini-2.5-flash",
      "fallbacks": ["ollama/qwen2:0.5b"]
    }
  }
}
```

## 7. Final Troubleshooting (The "Cold Reset")
If the bot says "IP Restricted" even with correct Google Cloud settings, it is usually a **cached session ghost**.

### To Perform a Cold Reset:
```bash
sudo systemctl stop openclaw
rm ~/.openclaw/agents/main/sessions/*.json

# (Restoring auth-profiles.json if deleted)
sudo systemctl start openclaw

clawdbot status    #show openclaw status
```

---
**Note:** Always verify your outbound IPv4 with `curl ifconfig.me` and ensure it matches your Google Cloud API whitelist.
