# Compute Configuration: Amazon EC2

This document details the setup, configuration, and operational management of the EC2 instance hosting the Flask application.

## 1. Instance Specifications
The compute layer is designed to fit entirely within the **AWS Free Tier**.

*   **Instance Type**: `t2.micro` (Variable, burstable performance) or `t3.micro`.
*   **AMI**: Amazon Linux 2023 (AL2023)
*   **Key Pair**: `ec2-server.pem` (RSA 2048-bit)
*   **User Data**: None (Manual provisioning used).

## 2. Setup & Provisioning Process

### Step 1: Launch & Network
1.  Launched into VPC `vpc-xxxxxxxxxxxxxxxxx`.
2.  Placed in Public Subnet `pubsub1`.
3.  Public IP Auto-assign: **Enabled**.
4.  Attached IAM Role: `EC2-S3-App-Role`.

### Step 2: Environment Setup
Commands executed after SSH connection:

```bash
# Update system
sudo yum update -y

# Install dependencies
sudo yum install git python3 python3-pip -y

# Clone Application
git clone https://github.com/Preygle/receipt_analyzer.git
cd receipt_analyzer

# Install App Dependencies
pip3 install -r requirements.txt
```

---

## 3. Operations & Persistence (Systemd)

To ensure the application is robust and auto-restarts on failure or reboot, a **systemd** service unit was created. This creates a daemon process for the Flask app.

**File Path**: `/etc/systemd/system/flaskapp.service`

```ini
[Unit]
Description=Flask App
# Start only after network is fully initialized
After=network.target

[Service]
# Run as the default EC2 user, not root (Security Best Practice)
User=ec2-user
WorkingDirectory=/home/ec2-user/receipt_analyzer

# Command to launch the app
ExecStart=/usr/bin/python3 /home/ec2-user/receipt_analyzer/app.py

# Critical: Restart automatically if the app crashes
Restart=always

[Install]
# Enable starting at boot time
WantedBy=multi-user.target
```

### Management Commands
```bash
# Reload changes
sudo systemctl daemon-reload

# Enable start on boot
sudo systemctl enable flaskapp

# Start immediately
sudo systemctl start flaskapp

# Check status/Logs
sudo systemctl status flaskapp
```

---

## 4. Monitoring
**Amazon CloudWatch** is configured to monitor instance health.

*   **Metric**: `CPUUtilization`
*   **Alarm**: `alarmname_96`
*   **Threshold**: Avg CPU > 70% for 5 minutes.
*   **Action**: Send notification via **Amazon SNS**.
