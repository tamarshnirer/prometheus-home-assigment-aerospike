# Prometheus Alertmanager Project

## Project Overview:
This project implements a monitoring and alerting solution using Prometheus and Alertmanager for an Amazon EC2 instance. It tracks CPU utilization and sends real-time alerts to a designated Slack channel if CPU usage exceeds 85%.

## Key Features:

- Monitoring: Prometheus collects metrics from the EC2 instance.
- Alerting: Alertmanager processes Prometheus alerts and triggers a Slack notification when the threshold is breached.
- Integration: Slack integration is done using a webhook and the alerts are delivered to the #alerts channel.

## Setup Instructions

### Prerequisites
- 2 Ubuntu servers with version 20.04 or higher (One for serving as a target and the other one for running Prometheus and alertmanager. 
- A Slack Workspace.

## Steps
### 1. Setting Up the Target:
 - Create an EC2 instance (e.g., t2.micro) or any virtual/physical machine running Ubuntu (e.g., 22.04).  
 - Attach an Elastic IP for a consistent public address.  
 - Configure a security group to allow:  
   - **Port 9100** (for metrics collection)  
   - **Port 22** (for SSH access).  
- Connect to the instance using SSH.
- Download Node Exporter using the following commands:
```bash
sudo apt update
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
tar xvfz node_exporter-1.8.2.linux-amd64.tar.gz
```
- Create a system user to run the service.
```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin node_exporter
sudo chown -R node_exporter:node_exporter /home/ubuntu/node_exporter-1.8.2.linux-amd64/
sudo apt install acl
sudo setfacl -m u:node_exporter:x /home/ubuntu/node_exporter-1.8.2.linux-amd64/node_exporter
sudo setfacl -m u:node_exporter:x /home/ubuntu/node_exporter-1.8.2.linux-amd64
```
- Set a service to run the Node Exporter.
```bash
vim /etc/systemd/system/node_exporter.service
```
```bash
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/home/ubuntu/node_exporter-1.8.2.linux-amd64/node_exporter

[Install]
WantedBy=multi-user.target
```
- Run the service, and make sure it's active by using the systemctl status command. 
```bash                           
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```
- Download the stress command for testing purposes:
```bash
sudo apt install -y stress
```
### 2. Creating a Slack webhook.
- Create a dedicated channel for the Prometheus notification in your Slack workspace. (e.g. #alerts)
- Create a Slack App by going to https://api.slack.com/apps
- Click "Create New App" and choose "From scratch"
- Name your app (e.g., "Prometheus Alerts") and select your workspace
- In your app's settings, find "Incoming Webhooks" in the features menu
- Toggle the switch to enable incoming webhooks
- Click "Add New Webhook to Workspace"
- Select the #alerts channel (or create it if it doesn't exist)
- Authorize the app to post to the selected channel
- Copy the webhook URL (it should look like https://hooks.slack.com/services/XXXX/XXXX/XXXX)
### 3. Setting Up the Monitoring Server
- Log in or SSH to your 2nd Ubuntu machine (I used Ubuntu 22.04 over wsl)
- Be sure ports 9090 and port 9093 are open and not used.
- Clone this repository on your Ubuntu machine.
```bash
git clone https://github.com/tamarshnirer/prometheus-alertmanager.git
cd prometheus-alertmanager
```
In the directory, you can spot 4 important files:  </br>
1. prometheus.yml - For configuring the targets and the alerting tool </br>
2. alert.rules.yml - For defining the rules and their severity levels </br>
3. alertmanager.yml - For defining the plugins the notifications will be sent to  </br>
4. docker-compose.yml - For configuring the prometheus and alertmanager containers </br>

- Create two environment variables: The public IP of the EC2 and the slack webhook. </br>
export TARGET_IP=<your_target_ip> </br>
export SLACK_API_WEBHOOK=<your_slack_webhook>  </br>
Note: Keep them secure! for large-scale production systems, it can be used by 3rd party vaults.
- Plugin those env var to the YAML files.
```bash
sed -i "s/TARGET_IP/$TARGET_IP/g" prometheus.yml
sed -i "s/SLACK_API_WEBHOOK/$SLACK_API_WEBHOOK/g" alertmanager.yml
```
- Create the persistent data volumes for the containers.
```bash
mkdir prometheus_data
mkdir alertmanager_data
```
- Download docker and docker-compose
```bash
sudo apt update
sudo apt install docker
sudo apt install docker-compose
```
- Run the docker-compose image. It should download the base images, build them, and run the containers in a bridge network mode.
```bash
docker-compose up -d
```
- You can see the Prometheus and Alertmanager UI in the following URLs accordingly: </br>
  http://localhost:9090/ </br>
  http://localhost:9093/

### 4. Testing the system
- Stress the target machine with a stress test, and plug in the number of vCPUs of the machine.
```bash
sudo stress --cpu 1 -v --timeout 300s
```
- You can view that the alert is being fired within 30 seconds.
- You should get a high CPU  alert to your Alets slack channel

<img width="950" alt="Screenshot 2024-11-21 174445" src="https://github.com/user-attachments/assets/459cce2b-528c-4539-96cb-ef0d86b0cdd8">
