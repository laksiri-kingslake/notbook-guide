# Jupyter Notebook Guide

A simple guide to install, configure and use Jupyter notebook effectively

## Step by Step Guide

### Installation

1. Update system
```bash
sudo apt update
sudo apt upgrade -y
```

2. Install Python venv tools
```bash
sudo apt install -y python3 python3-venv python3-pip build-essential

# (optional but useful)
sudo apt install -y ufw  # firewall
```

3. Create a dedicated user (optional but recommended):
```bash
# create user 'jupyter' (skip if you want to use existing user)
sudo adduser jupyter

# optionally give sudo to the user
# sudo usermod -aG sudo jupyter
```

4. Switch to that user (or use your current account):
```bash
# become that user
sudo -i -u jupyter bash
cd ~
```

5. Create & activate a virtual environment, then install Notebook:
```bash
python3 -m venv ~/venv
source ~/venv/bin/activate
pip install --upgrade pip
pip install notebook   # classic Jupyter Notebook

# optional: pip install jupyterlab  (if you prefer JupyterLab UI)
```

### Configure, Generate Password and Generate Config

1. Generate password

Still in the venv as the jupyter user:
```bash
# interactively set a password (preferred)
jupyter notebook password

# or generate a hashed password programmatically:
python3 - <<'PY'
from notebook.auth import passwd
print(passwd())   # type password when prompted; copy printed hash
PY
```

2. Generate the config file:
```bash
jupyter notebook --generate-config
# by default: ~/.jupyter/jupyter_notebook_config.py
```

3. Edit configuration file
```bash
nano ~/.jupyter/jupyter_notebook_config.py
```

Add/update these lines (examples)
```python
c = get_config()

c.NotebookApp.open_browser = False
c.NotebookApp.port = 8888
# Recommended for SSH-tunnel approach:
c.NotebookApp.ip = '127.0.0.1'
# If you really must expose publicly (NOT recommended without TLS), use:
# c.NotebookApp.ip = '0.0.0.0'

# If you used the hashed password approach, set:
# c.NotebookApp.password = u'sha1:...the-hash-you-copied...'

# If you want to use TLS (self-signed or valid cert):
# c.NotebookApp.certfile = u'/home/jupyter/.jupyter/mycert.pem'
# c.NotebookApp.keyfile  = u'/home/jupyter/.jupyter/mykey.key'
```

### Quick Test: Start Jupyter Manually

Activate venv then run:
```bash
source ~/venv/bin/activate
jupyter-notebook --no-browser --port=8888 --ip=127.0.0.1
```
**When it starts it prints a URL with a token. If you set a password, the web UI will ask for it**

### Run Jupyter as a Systemd Service (Recommended)

1 Create a systemd unit so Jupyter starts at boot. On the server (as root or sudo):
```bash
sudo tee /etc/systemd/system/jupyter.service > /dev/null <<'UNIT'
[Unit]
Description=Jupyter Notebook
After=network.target

[Service]
Type=simple
User=jupyter
Group=jupyter
WorkingDirectory=/home/jupyter
Environment="PATH=/home/jupyter/venv/bin"
ExecStart=/home/jupyter/venv/bin/jupyter-notebook --config=/home/jupyter/.jupyter/jupyter_notebook_config.py
Restart=on-failure

[Install]
WantedBy=multi-user.target
UNIT

# enable & start
sudo systemctl daemon-reload
sudo systemctl enable --now jupyter
sudo systemctl status jupyter
```
If you used a different username or path, update User=, WorkingDirectory=, PATH= and ExecStart= accordingly.

2. View logs:
```bash
sudo journalctl -u jupyter -f
```

### Access from Your Laptop (2 Secure Options)

#### Option 1 — SSH tunnel (fast & safest; no TLS required)

1. Port forward: On your laptop run
```bash
ssh -L 8888:localhost:8888 <your-server-user>@<server-ip-or-hostname>

# with .pem file
# ssh i <path-to-pem-file> -L 8888:localhost:8888 <your-server-user>@<server-ip-or-hostname>
```

2. Then open in your laptop browser:
```text
http://localhost:8888
```
You’ll be prompted for token/password. This tunnels your browser traffic securely over SSH; recommended if you only need to access from your laptop

#### Option 2 — Public HTTPS via nginx + Let’s Encrypt (use if you have a domain)

If you need access without SSH or from many clients, place nginx as a reverse proxy + TLS (Let’s Encrypt).

1. Install and configure nginx and certbot
```bash
# install nginx and certbot
sudo apt install -y nginx certbot python3-certbot-nginx

# example nginx site config /etc/nginx/sites-available/jupyter
sudo tee /etc/nginx/sites-available/jupyter > /dev/null <<'NGINX'
server {
    listen 80;
    server_name jupyter.example.com;

    location / {
        proxy_pass http://127.0.0.1:8888;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
NGINX

sudo ln -s /etc/nginx/sites-available/jupyter /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx

# obtain TLS cert (interactive)
sudo certbot --nginx -d jupyter.example.com

```
Important:
- In the Jupyter config set c.NotebookApp.ip = '127.0.0.1' (keep Jupyter bound to localhost so nginx handles external TLS).
- Protect Jupyter with a password (see earlier). Optionally restrict nginx with basic auth or firewall rules.

### Set-up Different Virtual Environments
Use different virtual environments for different notebooks in Jupyter. The trick is: each virtual environment needs to be registered as a Jupyter kernel. Then, inside Jupyter Notebook/Lab, you can select which environment to use for a given notebook.

1. Create your environments
```bash
# Torch env
python3 -m venv ~/envs/torch-env
source ~/envs/torch-env/bin/activate
pip install --upgrade pip
pip install torch jupyter ipykernel
deactivate

# Numpy/Pandas env
python3 -m venv ~/envs/data-env
source ~/envs/data-env/bin/activate
pip install --upgrade pip
pip install numpy pandas jupyter ipykernel
deactivate

```

2. Register each environment as a kernel
```bash
# Torch env
source ~/envs/torch-env/bin/activate
python -m ipykernel install --user --name=torch-env --display-name "Python (Torch)"
deactivate

# Data env
source ~/envs/data-env/bin/activate
python -m ipykernel install --user --name=data-env --display-name "Python (Data)"
deactivate

```
- --name → the internal kernel identifier
- --display-name → the friendly name you’ll see in Jupyter Notebook’s Kernel menu

3. Start Jupyter Notebook (in your base env)


### Firewall Rules (if using direct port)
If exposing port 8888 directly, restrict access to your laptop IP:
```bash
# example: allow only from your laptop's public IP
sudo ufw allow from <your-laptop-ip>/32 to any port 8888 proto tcp
# or open to everyone (not recommended):
# sudo ufw allow 8888/tcp
sudo ufw enable
sudo ufw status

```

### Useful commands & troubleshooting
- Start manually (inside venv): jupyter-notebook --no-browser --port=8888 --ip=127.0.0.1
- Service logs: sudo journalctl -u jupyter -f
- Check Jupyter config location: jupyter --path or config at ~/.jupyter/jupyter_notebook_config.py
- Show running notebook servers: jupyter notebook list

### Extras & recommendations
- If multiple users or multi-user production, use JupyterHub (designed for multiple user auth).
- Consider installing nbgrader, ipywidgets, or kernels you need 
```bash
pip install ipykernel and python -m ipykernel install --user --name myenv --display-name "Python (myenv)"
```
- Keep the server OS and packages updated. Don’t expose raw Jupyter over the public internet WITHOUT TLS and a strong password.