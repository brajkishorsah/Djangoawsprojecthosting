# AWS Django REST API Deployment Guide

This guide provides step-by-step instructions to deploy your Django REST API project on AWS (EC2 Ubuntu instance) using Gunicorn and Nginx.

---

## Step 1: Set Up EC2 Instance on AWS
1. Go to your AWS Console and launch an **EC2 Instance** (Ubuntu 22.04 LTS is recommended).
2. In the **Security Group**, open the following ports under **Inbound Rules**:
   - **Port 22** (SSH) - To connect from your Mac.
   - **Port 80** (HTTP) - For public internet access.
   - **Port 443** (HTTPS) - For secure API connections.
3. Start the instance and login via SSH:
   ```bash
   ssh -i "your-key.pem" ubuntu@your-server-ip
   ```

---

## Step 2: Update Server and Install Required Software
After logging into the server, run the following commands in the terminal:
```bash
sudo apt update
sudo apt upgrade -y
sudo apt install python3-pip python3-venv nginx git -y
```

---

## Step 3: Bring Your Project to the Server
You can clone your code to the server using GitHub:
```bash
git clone https://github.com/your-username/your-repo.git
cd your-repo
```

---

## Step 4: Create a Python Virtual Environment and Install Packages
Inside your project folder:
```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
pip install gunicorn  # Installing Gunicorn server is required
```

---

## Step 5: Update Environment Variables and Settings
Configure your `.env` or `settings.py` according to the production environment:
1. Create your `.env` file (`nano .env`) and add Database details, `SECRET_KEY`, etc.
2. Make sure to check these changes in your `settings.py`:
   ```python
   DEBUG = False
   ALLOWED_HOSTS = ['your-server-ip', 'your-domain.com']
   ```

Then set up migrations and static files:
```bash
python manage.py migrate
python manage.py collectstatic
```

---

## Step 6: Set Up Gunicorn (Background Server)
Gunicorn will keep your Django app running in the background. For this, we will create an Ubuntu systemd service.

Create the service file:
```bash
sudo nano /etc/systemd/system/gunicorn.service
```
Add these details (change the paths according to your project):
```ini
[Unit]
Description=gunicorn daemon
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/your-repo
ExecStart=/home/ubuntu/your-repo/venv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/home/ubuntu/your-repo/your_project.sock your_project_name.wsgi:application

[Install]
WantedBy=multi-user.target
```
*Note: `your_project_name` is the folder that contains your `wsgi.py` file.*

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

Start and enable Gunicorn:
```bash
sudo systemctl start gunicorn
sudo systemctl enable gunicorn
```

---

## Step 7: Set Up Nginx (Web Server)
Nginx will receive requests and forward them to Gunicorn.

Create an Nginx configuration file:
```bash
sudo nano /etc/nginx/sites-available/your_project
```
Add the following configuration:
```nginx
server {
    listen 80;
    server_name your-server-ip or your-domain.com;

    location = /favicon.ico { access_log off; log_not_found off; }
    
    # If using static files
    location /static/ {
        root /home/ubuntu/your-repo;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/ubuntu/your-repo/your_project.sock;
    }
}
```
Save and exit.

Now create a symlink of this file in `sites-enabled` and restart Nginx:
```bash
sudo ln -s /etc/nginx/sites-available/your_project /etc/nginx/sites-enabled
sudo nginx -t  # To check if there are any syntax errors
sudo systemctl restart nginx
```

---

**That's it! Your API is now hosted on AWS.** 
You can check it by entering your IP address in a browser or Postman. 

*(Optional but recommended: In the future, you can configure SSL/HTTPS by running `sudo apt install certbot python3-certbot-nginx` to secure your domain.)*
