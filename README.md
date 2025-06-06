Uptime Monitor with Uptime Kuma and Cloudflare
This project sets up a self-hosted status page and network monitoring service using Uptime Kuma. It runs in a Docker container on a Linux server and is securely exposed to the internet via a Cloudflare Tunnel, eliminating the need to open any firewall ports.

The primary goal of this setup is to monitor the individual uptime of a home network's router and modem to diagnose and pinpoint the source of internet connectivity issues.

Features
Beautiful Web UI: Clean, modern, and responsive status page.
Multi-Protocol Monitoring: Tracks uptime for HTTP(s), TCP, Ping, and more.
Individual Device Tracking: Configured to separately ping a local router and a reliable external DNS to determine where a network failure originates.
Secure Remote Access: Uses a Cloudflare Tunnel to securely expose the Uptime Kuma dashboard to a public domain without opening ports or exposing the server's IP address.
Real-time Notifications: Sends instant alerts via Discord, Email, and 90+ other notification services when a monitored service goes down.
Dockerized: The entire application stack is managed with Docker and Docker Compose for easy setup, teardown, and portability.
Tech Stack
Monitoring: Uptime Kuma
Containerization: Docker & Docker Compose
Ingress & Security: Cloudflare Tunnel
Setup and Installation
These instructions assume you have a Linux server with Docker and Docker Compose installed, and a domain name managed by Cloudflare.

1. Initial Setup
First, clone the repository or create the project directory structure.

Bash

mkdir -p uptime
cd uptime
2. Docker Compose Configuration
Create a docker-compose.yml file with the following content. This file defines two services: uptime-kuma for the application itself and cloudflared for the secure tunnel. They communicate over a shared Docker network.

YAML

# docker-compose.yml
version: '3.8'

services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    volumes:
      - ./uptime-kuma-data:/app/data
    networks:
      - uptime-network # Connect to the shared network
    restart: always

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared-tunnel
    restart: always
    command: tunnel --no-autoupdate run
    environment:
      - TUNNEL_TOKEN=${TUNNEL_TOKEN} # Read the token from the .env file
    networks:
      - uptime-network # Connect to the shared network

networks:
  uptime-network: # Define the shared network
3. Cloudflare Tunnel Setup
A Cloudflare Tunnel is required to securely expose the service.

Create the Tunnel: Follow the instructions on the Cloudflare Zero Trust Dashboard under Access > Tunnels to create a new tunnel.
Get the Token: After creating the tunnel, Cloudflare will provide a token. Create a file named .env in your project directory.
Configure the Token: Add your token to the .env file:
# .env
TUNNEL_TOKEN=YOUR_COPIED_TOKEN_HERE
Configure Public Hostname: In the tunnel's "Public Hostnames" configuration, create a record pointing to your Uptime Kuma container:
Subdomain: status (or your preferred subdomain)
Domain: yourdomain.com
Service Type: HTTP
URL: uptime-kuma:3001
4. Git Configuration
To prevent sensitive files and application data from being committed to the repository, create a .gitignore file.

# .gitignore

# Ignore the environment file containing the Cloudflare Tunnel token
.env

# Ignore the Uptime Kuma database and data files
uptime-kuma-data/
5. Launch the Application
With all the configuration files in place, launch the application stack:

Bash

docker-compose up -d
Usage
Access the Dashboard: Navigate to the public URL you configured (e.g., https://status.yourdomain.com).
Create an Admin Account: On your first visit, Uptime Kuma will prompt you to create an administrator account.
Set Up Monitors:
Router Monitor: Create a Ping monitor pointing to your router's local IP address (e.g., 192.168.1.1).
Internet/Modem Monitor: Create another Ping monitor pointing to a reliable external IP address (e.g., Google's DNS at 8.8.8.8).
Set Up Notifications:
Navigate to Settings > Notifications.
Click "Setup Notification" and choose your preferred service (e.g., Discord, Email).
Follow the prompts to add your credentials (like a Webhook URL or SMTP details).
Edit your monitors to enable the new notification channel.
Troubleshooting
Throughout the setup of this project, several common issues were resolved:

Git History and Sensitive Files: If .env files or data directories are accidentally committed, they must be removed from the repository's history using git rm --cached and a tool like git-filter-repo, not just deleted from the latest commit.
Diverged Git Branches: When making changes on multiple machines, branches can diverge. This is cleanly solved using git pull --rebase to replay local commits on top of remote changes.
Rebase Conflicts: Conflicts during a rebase (e.g., in .gitignore) must be resolved manually. Open the conflicted file, edit it to the final desired state, then use git add <file> and git rebase --continue.
Docker File Permissions: A Permission denied error during Git operations on volume-mounted data (uptime-kuma-data/) indicates the Docker container's user (root) owns the files, not the host user ($USER). The fix is to stop the containers (docker-compose down), change ownership (sudo chown -R $USER:$USER uptime-kuma-data/), and then perform the Git operations before starting the containers again (docker-compose up -d).
Cloudflare DNS Errors: An error like Error getting DNS record... when saving a tunnel hostname usually means a conflicting DNS record already exists. This is solved by manually deleting any existing A, AAAA, or CNAME records for that hostname in the main Cloudflare DNS dashboard before configuring the public hostname in the tunnel settings.
