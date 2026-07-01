# VPS Deployment Guide (DigitalOcean / Hostinger)

This guide walks you through deploying the SM Enterprises Shipment & Billing Portal to a **Virtual Private Server (VPS)** running **Ubuntu** (recommended version: 20.04 or 22.04 LTS).

Deploying on a VPS gives you 100% control over the environment and provides built-in persistent disk storage for your SQLite database.

---

## 📋 Prerequisites
- An active account on **DigitalOcean** (create a "Droplet") or **Hostinger** (create an "Ubuntu VPS").
- A domain name (e.g., `smenterprisesagra.com`) pointing to your VPS IP address (set up an **A record** in your domain registrar DNS).

---

## 🚀 Step 1: SSH into your VPS
Open PowerShell (Windows) or Terminal (macOS/Linux) and connect to your server using its IP address:
```bash
ssh root@YOUR_VPS_IP_ADDRESS
```
Enter your root password when prompted.

---

## 🛠️ Step 2: System Setup (Node.js & SQLite)
Update the packages list and install Node.js (version 18+), Git, and build tools:
```bash
# Update Ubuntu package registry
sudo apt update && sudo apt upgrade -y

# Install Git and Curl
sudo apt install -y git curl build-essential

# Download and install Node.js v18 LTS
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
```
Verify the installation by running:
```bash
node -v
npm -v
```

---

## 📦 Step 3: Clone Code and Configure Environment
1. Clone your private GitHub repository into the `/var/www` directory:
   ```bash
   cd /var/www
   git clone https://github.com/your-username/sm-enterprises-portal.git smenterprises
   cd smenterprises
   ```
2. Install the production dependencies:
   ```bash
   npm install --production
   ```
3. Create your production environment file:
   ```bash
   nano .env
   ```
   Add the following variables inside nano editor (press `CTRL+O` then `Enter` to save, and `CTRL+X` to exit):
   ```env
   PORT=3000
   ADMIN_PASSWORD=your_secure_dashboard_password_here
   ```

---

## 🔄 Step 4: Run the App in the Background with PM2
PM2 is a production process manager that keeps your Node.js application running 24/7, restarts it automatically if it crashes, and starts it up on server boot.

1. Install PM2 globally:
   ```bash
   sudo npm install -g pm2
   ```
2. Start your application:
   ```bash
   pm2 start server.js --name "sm-portal"
   ```
3. Configure PM2 to start automatically on system reboots:
   ```bash
   pm2 startup systemd
   ```
   *(Copy and paste the command output by PM2 into your terminal to activate startup).*
4. Save the active process list:
   ```bash
   pm2 save
   ```

---

## 🌐 Step 5: Configure Nginx as a Reverse Proxy
Nginx acts as a high-performance web server that routes incoming internet requests (port 80) to your Node.js application running on port 3000.

1. Install Nginx:
   ```bash
   sudo apt install -y nginx
   ```
2. Create a configuration file for your website:
   ```bash
   sudo nano /etc/nginx/sites-available/smenterprises
   ```
3. Paste the following configuration, replacing `yourdomain.com` with your actual domain name:
   ```nginx
   server {
       listen 80;
       server_name yourdomain.com www.yourdomain.com;

       location / {
           proxy_pass http://localhost:3000;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host $host;
           proxy_cache_bypass $http_upgrade;
       }
   }
   ```
4. Enable the site configuration:
   ```bash
   sudo ln -s /etc/nginx/sites-available/smenterprises /etc/nginx/sites-enabled/
   # Remove default nginx config to prevent conflicts
   sudo rm /etc/nginx/sites-enabled/default
   ```
5. Test the configuration and restart Nginx:
   ```bash
   sudo nginx -t
   sudo systemctl restart nginx
   ```

---

## 🔒 Step 6: Install SSL (HTTPS) with Let's Encrypt
To secure the admin dashboard and enable HTTPS, we use Certbot to get a free, auto-renewing SSL certificate.

1. Install Certbot:
   ```bash
   sudo apt install -y certbot python3-certbot-nginx
   ```
2. Obtain the certificate:
   ```bash
   sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
   ```
3. Follow the interactive prompts (enter your email, accept the terms). Certbot will automatically rewrite your Nginx configuration to force HTTPS redirection.
4. Verify auto-renewal:
   ```bash
   sudo certbot renew --dry-run
   ```

---

## 🗄️ Database Backups (Recommended)
Since SQLite stores everything in the `/var/www/smenterprises/shipments.db` file, taking backups is simple. You can copy this file using SFTP (via FileZilla) or set up a cron job to zip it daily.
