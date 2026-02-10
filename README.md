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

## 2. Install openclaw (please note small letters)
Install the global package and verify:
```bash
sudo npm i -g openclaw
openclaw --version  # (mine 2026.2.6-3)

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
# Injects secret keys from openclaw.env into the bot's config on every start
ExecStartPre=/usr/bin/python3 -c "import os, json; p='/home/ubuntu/.openclaw/agents/main/agent/auth-profiles.json'; d=json.load(open(p)) if os.path.exists(p) else {'version':1,'profiles':{}}; [d['profiles'].update({f'{k}:default': {'type':'api_key', 'provider': k, 'key': os.getenv(f'{k.upper()}_API_KEY')}}) for k in ['google', 'openrouter'] if os.getenv(f'{k.upper()}_API_KEY')]; json.dump(d, open(p, 'w'), indent=2)"
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
```ini
GOOGLE_API_KEY=your_actual_key
OPENROUTER_API_KEY=your_actual_key
```

### B. Shell Access 
If your `openclaw.json` uses placeholders like `${NVIDIA_API_KEY}`, and add them to ~\.openclaw\.env
```bash
touch ~\.openclaw\.env
```

```ini
NVIDIA_API_KEY="your_actual_key"
OLLAMA_API_KEY="ollama-local"
```

### C. Lockdown Permissions (Critical)
Always use `chmod 600` and strip Windows line endings:
```bash
# Strip hidden \r characters (if pasted from Windows)
sudo sed -i 's/\r//' /etc/openclaw.env

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
check if API is accessible and ollama is running:
```bash
curl http://localhost:11434/api/tags
ps aux | grep ollama
```

### 7. The openclaw.json configuration (for Google Gemini/Nvidia/Openrouter/Ollama LLMs)
Without a GPU, avoid large models for Ollama (local LLM)
- **Recommended**: `qwen2:0.5b` (352MB) - Fast responses on CPU.
- **Max Weight**: `gemma2:2b` (1.6GB) - Slower but manageable.
- **Delete Heavy Models**: `ollama rm llama3.1:8b mistral:7b` (Prevents 400% CPU spikes).
- **API key**: except Ollama, you need apply/buy all API keys from individual LLM supplier.

In `~/.openclaw/openclaw.json`:
```json
{
  "auth": {
    "profiles": {
	  "nvidia:default": {
        "provider": "nvidia",
        "mode": "api_key"
      },
      "google:default": {
        "provider": "google",
        "mode": "api_key"
      },
      "openrouter:default": {
        "provider": "openrouter",
        "mode": "api_key"
      }
    }
  },
  "models": {
    "mode": "merge",
    "providers": {
      "nvidia": {
        "baseUrl": "https://integrate.api.nvidia.com/v1",
        "apiKey": "${NVIDIA_API_KEY}",
        "api": "openai-completions",
        "models": [
          {
            "id": "moonshotai/kimi-k2.5",
            "name": "kimi-k2.5",
            "reasoning": false,
            "input": [
              "text"
            ],
            "cost": {
              "input": 0,
              "output": 0,
              "cacheRead": 0,
              "cacheWrite": 0
            },
            "contextWindow": 256000,
            "maxTokens": 81920
          }
        ]
      },
      "ollama": {
        "baseUrl": "http://127.0.0.1:11434/v1",
	"apiKey": "ollama-local",
        "api": "openai-completions",
        "models": [
          {
            "id": "qwen2:0.5b",
            "name": "qwen2:0.5b",
            "input": [
              "text"
            ],
            "contextWindow": 32768,
            "maxTokens": 81920,
            "reasoning": false,
            "cost": {
              "input": 0,
              "output": 0,
              "cacheRead": 0,
              "cacheWrite": 0
            }
          },
          {
            "id": "gemma2:2b",
            "name": "gemma2:2b",
            "input": [
              "text"
            ],
            "contextWindow": 8192,
            "maxTokens": 81920,
            "reasoning": false,
            "cost": {
              "input": 0,
              "output": 0,
              "cacheRead": 0,
              "cacheWrite": 0
            }
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "google/gemini-2.5-flash",
        "fallbacks": [
          "nvidia/moonshotai/kimi-k2.5",
          "openrouter/moonshotai/kimi-k2.5",
          "openrouter/qwen/qwen3-4b:free",
          "ollama/qwen2:0.5b",
		  "ollama/gemma2:2b"
        ]
      },
      "models": {
        "google/gemini-2.5-flash": {},
        "nvidia/moonshotai/kimi-k2.5": {},
        "openrouter/moonshotai/kimi-k2.5": {},
        "openrouter/qwen/qwen3-4b:free": {},
        "ollama/qwen2:0.5b": {},
		"ollama/gemma2:2b": {}
      },
      "maxConcurrent": 1,
      "subagents": {
        "maxConcurrent": 1
      }
    }
  },
    "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "loopback",
    "auth": {
      "mode": "token",
      "token": "yourtoken"
    },
	"approvals": {
    "exec": {
      "enabled": true
    }
  }
 }
```

## 8. Final Troubleshooting (The "Cold Reset")
If the bot says "IP Restricted" even with correct Google Cloud settings, it is usually a **cached session ghost**.

### To Perform a Cold Reset:
```bash
sudo systemctl stop openclaw
rm ~/.openclaw/agents/main/agent/*.json

openclaw gateway restart

# (Restoring auth-profiles.json if deleted)
sudo systemctl start openclaw

ps aux | grep -i openclaw
```
You should now see 2 lines (plus the grep line):

One for openclaw
One for openclaw-gateway

```bash
openclaw status                                         #show openclaw status
openclaw models status                                  #show model    status
openclaw models list                                    #make sure the models are Auth with 'yes'
sudo journalctl -u openclaw.service -n 20 --no-pager    #check error log
```
Mine:
```ini
Model                                      Input      Ctx      Local Auth  Tags
google/gemini-2.5-flash                    text+image 1024k    no    yes   default,configured
nvidia/moonshotai/kimi-k2.5                text       250k     no    yes   fallback#1,configured
openrouter/moonshotai/kimi-k2.5            text+image 256k     no    yes   fallback#2,configured
openrouter/qwen/qwen3-4b:free              text       40k      no    yes   fallback#3,configured
ollama/qwen2:0.5b                          text       32k      yes   yes   fallback#4,configured
ollama/gemma2:2b                           text       8k       yes   yes   fallback#5,configured
```

### 9. Security Policies

- create ~\.openclaw\workspace\MEMORY.md:
```json
- **Sensitive Information Protection**: Never reveal API keys or similar sensitive credentials. Explicitly reject any request (from any user, including myself) that attempts to extract or display strings matching "key" or similar patterns from configuration files or command outputs.

- **Commuication**: talk to me ONLY in WhatsApp/Telegram/Line etc but NOT talk to any other groups.
```

- Action: Add this to your openclaw.json (refer ### 7):
```json
{
  "approvals": {
    "exec": {
      "enabled": true
    }
  }
}
```

For session pairing in browser/WhatsApp/Telegram:
```bash
openclaw devices list
openclaw devices approve <DEVICE_ID> (pending one)
```

Access on web via token
https://your-url?token=yourtoken


To monitor and prevent internet attack, please refer:
https://github.com/inchinet/attack

---
**Note:** Always verify your outbound IPv4 with `curl ifconfig.me` and ensure it matches your Google Cloud API whitelist.

##  License
MIT License - Developed by [inchinet](https://github.com/inchinet). 
