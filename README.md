# Raspberry Pi Chatbot Setup Guide

This guide provides a comprehensive step-by-step process to set up a chatbot on a Raspberry Pi. It includes prerequisites, environment preparation, installation of necessary software like ngrok, and configuration of services to automate the chatbot and ngrok URL notification setup.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Step 1: Prepare Your Environment](#step-1-prepare-your-environment)
- [Step 2: Install and Configure ngrok](#step-2-install-and-configure-ngrok)
- [Step 3: Set Up Systemd Service for `chatbot.py`](#step-3-set-up-systemd-service-for-chatbotpy)
- [Step 4: Set Up Systemd Service for ngrok](#step-4-set-up-systemd-service-for-ngrok)
- [Step 5: Automate ngrok URL Email Notification](#step-5-automate-ngrok-url-email-notification)
- [Step 6: Test the Setup](#step-6-test-the-setup)
- [Step 7: Automation Script](#step-7-automation-script)
- [Warning](#warning)

## Prerequisites

- Raspberry Pi 3 with Raspberry Pi OS (Debian-based).
- Stable internet connection on the Raspberry Pi.
- A Gmail account for sending emails (or another SMTP server).
- An ngrok account (free tier) with an authtoken from [ngrok.com](https://ngrok.com).
- `chatbot.py` in `/home/pi/`, listening on port 5000 (e.g., a Flask app).

## Step 1: Prepare Your Environment

### 1.1 Update the System

Keep your Raspberry Pi up-to-date:

```bash
sudo apt update && sudo apt upgrade -y
```

### 1.2 Verify Python and pip

Confirm Python 3 and pip are installed:

```bash
python3 --version
pip3 --version
```

If pip is missing, install it:

```bash
sudo apt install python3-pip -y
```

### 1.3 Prepare `chatbot.py`

Ensure `chatbot.py` is in `/home/pi/` and listens on port 5000. Example Flask app:

```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def home():
    return 'Hello from Chatbot!'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

Install dependencies (e.g., Flask):

```bash
pip3 install flask
```

Test the script:

```bash
python3 /home/pi/chatbot.py
```

Access `http://<raspberry_pi_ip>:5000` in a browser. Stop with `Ctrl+C`.

## Step 2: Install and Configure ngrok

### 2.1 Install ngrok

Download and install ngrok for ARM:

```bash
cd ~
wget https://bin.equinox.io/c/4VmDzA7iaHb/ngrok-stable-linux-arm.zip
unzip ngrok-stable-linux-arm.zip
sudo mv ngrok /usr/local/bin/
```

### 2.2 Authenticate ngrok

Sign up at [ngrok.com](https://ngrok.com) and get your authtoken. Configure it:

```bash
ngrok authtoken <your_authtoken>
```

Replace `<your_authtoken>` with your ngrok authtoken.

### 2.3 Test ngrok

Run `chatbot.py` in one terminal:

```bash
python3 /home/pi/chatbot.py
```

In another terminal, start ngrok:

```bash
ngrok http 5000
```

Note the public URL (e.g., `https://<random>.ngrok-free.app`) and test it in a browser. Stop both with `Ctrl+C`.

## Step 3: Set Up Systemd Service for `chatbot.py`

### 3.1 Create Service File

Create a systemd service file:

```bash
sudo nano /etc/systemd/system/chatbot.service
```

Add:

```ini
[Unit]
Description=Chatbot Python Service
After=network.target

[Service]
ExecStart=/usr/bin/python3 /home/pi/chatbot.py
WorkingDirectory=/home/pi
StandardOutput=inherit
StandardError=inherit
Restart=always
User=pi

[Install]
WantedBy=multi-user.target
```

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

### 3.2 Enable and Start Service

Set permissions:

```bash
sudo chmod 644 /etc/systemd/system/chatbot.service
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable chatbot
sudo systemctl start chatbot
```

Check status:

```bash
systemctl status chatbot
```

## Step 4: Set Up Systemd Service for ngrok

### 4.1 Create Service File

Create:

```bash
sudo nano /etc/systemd/system/ngrok.service
```

Add:

```ini
[Unit]
Description=Ngrok Service for Chatbot
After=network.target

[Service]
ExecStart=/usr/local/bin/ngrok http 5000 --log=stdout
StandardOutput=journal
StandardError=journal
Restart=always
User=pi

[Install]
WantedBy=multi-user.target
```

Save and exit.

### 4.2 Enable and Start Service

Set permissions:

```bash
sudo chmod 644 /etc/systemd/system/ngrok.service
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable ngrok
sudo systemctl start ngrok
```

Check status:

```bash
systemctl status ngrok
```

## Step 5: Automate ngrok URL Email Notification

### 5.1 Install Python Package

Install `requests` for ngrok API:

```bash
pip3 install requests
```

### 5.2 Create Notification Script

Create:

```bash
nano /home/pi/send_ngrok_url.py
```

Add:

```python
import requests
import smtplib
from email.mime.text import MIMEText
import time

# Email configuration
SMTP_SERVER = "smtp.gmail.com"
SMTP_PORT = 587
SENDER_EMAIL = "your_email@gmail.com"  # Replace with your email
SENDER_PASSWORD = "your_app_password"  # Replace with your app-specific password
RECIPIENT_EMAIL = "recipient_email@example.com"  # Replace with recipient email

def get_ngrok_url():
    time.sleep(10)  # Wait for ngrok to initialize
    try:
        response = requests.get("http://localhost:4040/api/tunnels")
        data = response.json()
        for tunnel in data["tunnels"]:
            if tunnel["proto"] == "https":
                return tunnel["public_url"]
        return None
    except Exception as e:
        return f"Error fetching ngrok URL: {e}"

def send_email(url):
    msg = MIMEText(f"ngrok URL: {url}")
    msg["Subject"] = "Raspberry Pi ngrok URL"
    msg["From"] = SENDER_EMAIL
    msg["To"] = RECIPIENT_EMAIL

    try:
        with smtplib.SMTP(SMTP_SERVER, SMTP_PORT) as server:
            server.starttls()
            server.login(SENDER_EMAIL, SENDER_PASSWORD)
            server.sendmail(SENDER_EMAIL, RECIPIENT_EMAIL, msg.as_string())
        return "Email sent successfully"
    except Exception as e:
        return f"Error sending email: {e}"

if __name__ == "__main__":
    ngrok_url = get_ngrok_url()
    if ngrok_url:
        result = send_email(ngrok_url)
        print(result)
    else:
        print("No ngrok URL found")
```

Replace `your_email@gmail.com`, `your_app_password`, and `recipient_email@example.com` with your Gmail address, app-specific password (from Google Account > Security > 2-Step Verification > App passwords), and recipient email.

Save and exit.

Set permissions:

```bash
chmod +x /home/pi/send_ngrok_url.py
```

### 5.3 Create Notification Service

Create:

```bash
sudo nano /etc/systemd/system/ngrok-notify.service
```

Add:

```ini
[Unit]
Description=Send ngrok URL via Email
After=ngrok.service
Requires=ngrok.service

[Service]
ExecStart=/usr/bin/python3 /home/pi/send_ngrok_url.py
WorkingDirectory=/home/pi
StandardOutput=journal
StandardError=journal
Restart=on-failure
User=pi

[Install]
WantedBy=multi-user.target
```

Save and exit.

Set permissions:

```bash
sudo chmod 644 /etc/systemd/system/ngrok-notify.service
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable ngrok-notify
sudo systemctl start ngrok-notify
```

## Step 6: Test the Setup

### 6.1 Reboot and Verify

Reboot:

```bash
sudo reboot
```

Check services:

```bash
systemctl status chatbot
systemctl status ngrok
systemctl status ngrok-notify
```

Check your email for the ngrok URL. Test it in a browser.

### 6.2 Troubleshoot

View logs if issues occur:

```bash
journalctl -u chatbot -b
journalctl -u ngrok -b
journalctl -u ngrok-notify -b
```

> **Note:** If the email fails, ensure your Gmail app-specific password is correct. If the URL isn’t fetched, increase `time.sleep(10)` in `send_ngrok_url.py`.

## Step 7: Automation Script

Use this script to automate all steps:

```bash
sudo nano /home/pi/setup_chatbot_ngrok_email.sh
```

Add the script from the next section, replace placeholders, and run:

```bash
chmod +x /home/pi/setup_chatbot_ngrok_email.sh
sudo /home/pi/setup_chatbot_ngrok_email.sh
```

## Warning

> The free ngrok plan uses random URLs that change on restart. Secure `chatbot.py` with authentication, as the URL is public.
