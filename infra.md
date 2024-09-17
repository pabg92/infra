## **Overview**

**Server Specification: Hetzner CX42**

- **vCPU**: 8
- **RAM**: 16 GB
- **Storage**: 160 GB NVMe SSD
- **Bandwidth**: 20 TB
- **Cost**: Approximately €29.27 (~$31) per month

**Budget Consideration**: While the CX42 exceeds your initial €25/month budget slightly, the increased resources provide significant additional capabilities that can enhance your portfolio and support complex projects.

---

## **Objectives**

1. **Deploy Vault Warden** accessible via the web.
2. **Set up GitHub/VSCode integration** to push changes to the live website on request.
3. **Implement a robust backup system** for data security.
4. **Add Monitoring and Logging** systems for performance tracking and troubleshooting.
5. **Include Additional Projects** to enhance your portfolio:
   - **CI/CD Pipeline** using Jenkins or GitLab CI.
   - **Self-hosted Git Service** with Gitea or GitLab.
   - **Nextcloud** for personal cloud storage.
   - **Infrastructure as Code (IaC)** using Ansible or Terraform.
   - **Documentation Site** using MkDocs or Hugo.
   - **User Management System** with Keycloak.
6. **Optimize Resource Utilization** to maximize server capabilities.
7. **Enhance Security** across all services.

---

## **Detailed Plan**

### **1. Server Setup and Initial Configuration**

#### **1.1. Choose Operating System**

- **Ubuntu 22.04 LTS**: Preferred for its stability and extensive community support.

#### **1.2. Initial Server Configuration**

1. **Access the Server via SSH**

   ```bash
   ssh root@your_server_ip
   ```

2. **Update and Upgrade Packages**

   ```bash
   apt update && apt upgrade -y
   ```

3. **Set Hostname and Timezone**

   ```bash
   hostnamectl set-hostname yourserverhostname
   timedatectl set-timezone Your/Timezone
   ```

4. **Create a New User with Sudo Privileges**

   ```bash
   adduser yourusername
   usermod -aG sudo yourusername
   ```

5. **Configure SSH Access**

   - **Disable Root Login and Password Authentication**

     ```bash
     nano /etc/ssh/sshd_config
     ```

     - Set the following parameters:

       ```
       PermitRootLogin no
       PasswordAuthentication no
       ```

   - **Restart SSH Service**

     ```bash
     systemctl restart sshd
     ```

6. **Set Up Firewall with UFW**

   ```bash
   apt install ufw -y
   ufw default deny incoming
   ufw default allow outgoing
   ufw allow ssh
   ufw allow 80
   ufw allow 443
   ufw enable
   ```

---

### **2. Install Docker and Docker Compose**

Docker is essential for containerizing applications, ensuring isolation, scalability, and ease of management.

#### **2.1. Install Docker**

```bash
sudo apt install apt-transport-https ca-certificates curl gnupg lsb-release -y
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io -y
```

- **Verify Docker Installation**

  ```bash
  sudo docker run hello-world
  ```

#### **2.2. Install Docker Compose**

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/v2.21.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

- **Verify Docker Compose Installation**

  ```bash
  docker-compose --version
  ```

#### **2.3. Manage Docker as a Non-Root User**

```bash
sudo usermod -aG docker yourusername
newgrp docker
```

---

### **3. Set Up a Reverse Proxy with Nginx Proxy Manager**

Managing multiple web services with SSL is simplified using Nginx Proxy Manager, which provides a user-friendly web interface.

#### **3.1. Create a Docker Network**

```bash
sudo docker network create proxy
```

#### **3.2. Deploy Nginx Proxy Manager via Docker Compose**

1. **Create Directory and Docker Compose File**

   ```bash
   mkdir ~/nginx-proxy-manager && cd ~/nginx-proxy-manager
   nano docker-compose.yml
   ```

   **Contents:**

   ```yaml
   version: '3'
   services:
     app:
       image: 'jc21/nginx-proxy-manager:latest'
       restart: always
       ports:
         - '80:80'
         - '81:81'  # Admin interface
         - '443:443'
       environment:
         DB_SQLITE_FILE: "/data/database.sqlite"
       volumes:
         - ./data:/data
         - ./letsencrypt:/etc/letsencrypt
       networks:
         - proxy
   networks:
     proxy:
       external: true
   ```

2. **Start Nginx Proxy Manager**

   ```bash
   sudo docker-compose up -d
   ```

#### **3.3. Access and Configure Nginx Proxy Manager**

- **Access the Web Interface**: `http://your_server_ip:81`
- **Default Credentials**:
  - **Email**: `admin@example.com`
  - **Password**: `changeme`
- **Update Credentials** upon first login.

---

### **4. Deploy Core Services**

#### **4.1. Deploy Vault Warden**

Vault Warden is a lightweight, self-hosted password manager compatible with Bitwarden clients.

1. **Create Directory and Docker Compose File**

   ```bash
   mkdir ~/vaultwarden && cd ~/vaultwarden
   nano docker-compose.yml
   ```

   **Contents:**

   ```yaml
   version: '3'
   services:
     vaultwarden:
       image: vaultwarden/server:latest
       container_name: vaultwarden
       restart: always
       volumes:
         - ./data:/data
       environment:
         WEBSOCKET_ENABLED: 'true'
       networks:
         - proxy
   networks:
     proxy:
       external: true
   ```

2. **Start Vault Warden**

   ```bash
   sudo docker-compose up -d
   ```

3. **Configure Reverse Proxy in Nginx Proxy Manager**

   - **Add a New Proxy Host**:
     - **Domain Names**: `vault.yourdomain.com`
     - **Scheme**: `http`
     - **Forward Hostname / IP**: `vaultwarden`
     - **Forward Port**: `80`
     - **Websockets Support**: Enabled
     - **SSL**: Enable and request an SSL certificate via Let's Encrypt.

4. **DNS Configuration**

   - Point `vault.yourdomain.com` to your server's IP address.

5. **Access Vault Warden**

   - Visit `https://vault.yourdomain.com` to access your Vault Warden instance.

---

#### **4.2. Deploy Gitea (Self-hosted Git Service)**

Gitea is a lightweight, self-hosted Git service ideal for managing repositories privately.

1. **Create Directory and Docker Compose File**

   ```bash
   mkdir ~/gitea && cd ~/gitea
   nano docker-compose.yml
   ```

   **Contents:**

   ```yaml
   version: '3'
   services:
     gitea:
       image: gitea/gitea:latest
       container_name: gitea
       environment:
         - USER_UID=1000
         - USER_GID=1000
       volumes:
         - ./gitea:/data
       networks:
         - proxy
   networks:
     proxy:
       external: true
   ```

2. **Start Gitea**

   ```bash
   sudo docker-compose up -d
   ```

3. **Configure Reverse Proxy in Nginx Proxy Manager**

   - **Add a New Proxy Host**:
     - **Domain Names**: `git.yourdomain.com`
     - **Scheme**: `http`
     - **Forward Hostname / IP**: `gitea`
     - **Forward Port**: `3000` (Ensure Gitea is configured to use port 3000)
     - **SSL**: Enable and request an SSL certificate via Let's Encrypt.

4. **DNS Configuration**

   - Point `git.yourdomain.com` to your server's IP address.

5. **Access Gitea**

   - Visit `https://git.yourdomain.com` to access your Gitea instance.
   - Complete the initial setup by following the on-screen instructions.

---

#### **4.3. Deploy Jenkins (CI/CD Pipeline)**

Jenkins automates the process of building, testing, and deploying your software projects.

1. **Create Directory and Docker Compose File**

   ```bash
   mkdir ~/jenkins && cd ~/jenkins
   nano docker-compose.yml
   ```

   **Contents:**

   ```yaml
   version: '3'
   services:
     jenkins:
       image: jenkins/jenkins:lts
       container_name: jenkins
       user: root
       restart: always
       ports:
         - "8080:8080"
         - "50000:50000"
       volumes:
         - ./jenkins_home:/var/jenkins_home
       networks:
         - proxy
   networks:
     proxy:
       external: true
   ```

2. **Start Jenkins**

   ```bash
   sudo docker-compose up -d
   ```

3. **Configure Reverse Proxy in Nginx Proxy Manager**

   - **Add a New Proxy Host**:
     - **Domain Names**: `ci.yourdomain.com`
     - **Scheme**: `http`
     - **Forward Hostname / IP**: `jenkins`
     - **Forward Port**: `8080`
     - **SSL**: Enable and request an SSL certificate via Let's Encrypt.

4. **DNS Configuration**

   - Point `ci.yourdomain.com` to your server's IP address.

5. **Access Jenkins**

   - Visit `https://ci.yourdomain.com` to access Jenkins.
   - Follow the initial setup wizard:
     - Retrieve the initial admin password:

       ```bash
       sudo cat ~/jenkins/jenkins_home/secrets/initialAdminPassword
       ```

     - Install suggested plugins and create an admin user.

6. **Integrate Jenkins with Gitea**

   - **Install Gitea Plugin in Jenkins**:
     - Navigate to **Manage Jenkins > Manage Plugins > Available**.
     - Search for **Gitea Plugin** and install it.

   - **Configure Gitea in Jenkins**:
     - **Manage Jenkins > Configure System > Gitea**.
     - Add your Gitea server details.

   - **Set Up Webhooks in Gitea**:
     - For each repository, navigate to **Settings > Webhooks**.
     - Add a new webhook pointing to `https://ci.yourdomain.com/gitea-webhook/`.

   - **Create Jenkins Jobs**:
     - Set up pipeline jobs that trigger on commits to your Gitea repositories.
     - Configure build steps, tests, and deployment scripts as needed.

---

#### **4.4. Deploy Your Website**

Depending on your website's technology stack, you can containerize it or host it directly on the server.

**Example: Hosting a Static Website with Nginx**

1. **Create Directory and Docker Compose File**

   ```bash
   mkdir ~/website && cd ~/website
   nano docker-compose.yml
   ```

   **Contents:**

   ```yaml
   version: '3'
   services:
     web:
       image: nginx:latest
       container_name: website
       volumes:
         - ./html:/usr/share/nginx/html:ro
       networks:
         - proxy
       restart: always
   networks:
     proxy:
       external: true
   ```

2. **Add Your Website Files**

   ```bash
   mkdir html
   # Upload your website files to the 'html' directory
   ```

3. **Start the Website Container**

   ```bash
   sudo docker-compose up -d
   ```

4. **Configure Reverse Proxy in Nginx Proxy Manager**

   - **Add a New Proxy Host**:
     - **Domain Names**: `www.yourdomain.com` and `yourdomain.com`
     - **Scheme**: `http`
     - **Forward Hostname / IP**: `website`
     - **Forward Port**: `80`
     - **Websockets Support**: Disabled
     - **SSL**: Enable and request an SSL certificate via Let's Encrypt.

5. **DNS Configuration**

   - Point `www.yourdomain.com` and `yourdomain.com` to your server's IP address.

6. **Access Your Website**

   - Visit `https://yourdomain.com` to view your live website.

**Note**: If your website uses a different stack (e.g., Node.js, Django), adjust the Docker setup accordingly.

---

### **5. Implement a Robust Backup System**

Ensuring data integrity and availability is crucial. We'll set up automated backups for all critical services.

#### **5.1. Directory Structure for Backups**

```bash
mkdir -p ~/backups/{vaultwarden,gitea,jenkins,website,nextcloud}
```

#### **5.2. Create a Backup Script**

1. **Create the Backup Script**

   ```bash
   sudo nano /usr/local/bin/backup.sh
   ```

2. **Script Contents**

   ```bash
   #!/bin/bash

   TIMESTAMP=$(date +"%F_%H-%M-%S")
   BACKUP_DIR="/home/yourusername/backups"

   # Function to backup Docker volumes
   backup_docker_volume() {
       CONTAINER_NAME=$1
       SERVICE_NAME=$2
       docker run --rm \
           --volumes-from $CONTAINER_NAME \
           -v $BACKUP_DIR:/backup \
           ubuntu bash -c "tar czvf /backup/${SERVICE_NAME}_$TIMESTAMP.tar.gz /data"
   }

   # Backup Vault Warden
   backup_docker_volume "vaultwarden" "vaultwarden"

   # Backup Gitea
   backup_docker_volume "gitea" "gitea"

   # Backup Jenkins
   tar czvf $BACKUP_DIR/jenkins_$TIMESTAMP.tar.gz /home/yourusername/jenkins/jenkins_home

   # Backup Website Files
   tar czvf $BACKUP_DIR/website_$TIMESTAMP.tar.gz /home/yourusername/website/html

   # (Repeat for additional services like Nextcloud)

   # Cleanup old backups (older than 7 days)
   find $BACKUP_DIR -type f -mtime +7 -name '*.tar.gz' -exec rm {} \;
   ```

   **Notes**:
   - Adjust paths as necessary.
   - Add backup commands for additional services like Nextcloud.

3. **Make the Script Executable**

   ```bash
   sudo chmod +x /usr/local/bin/backup.sh
   ```

#### **5.3. Schedule Backups with Cron**

1. **Edit Crontab**

   ```bash
   sudo crontab -e
   ```

2. **Add Cron Job**

   - To run the backup daily at 2 AM:

     ```cron
     0 2 * * * /usr/local/bin/backup.sh
     ```

#### **5.4. Implement Off-site Backups with Rclone**

Storing backups off-site enhances data security.

1. **Install Rclone**

   ```bash
   curl https://rclone.org/install.sh | sudo bash
   ```

2. **Configure Rclone**

   ```bash
   rclone config
   ```

   - **Steps**:
     - **n**: New remote
     - **Name**: e.g., `backupremote`
     - **Storage Type**: Choose your preferred cloud provider (e.g., Google Drive, AWS S3, Backblaze B2)
     - **Follow the prompts** to authenticate and configure.

3. **Modify Backup Script to Sync Backups**

   - **Edit the Backup Script**

     ```bash
     sudo nano /usr/local/bin/backup.sh
     ```

   - **Add Sync Command at the End**

     ```bash
     # Sync backups to remote
     rclone copy /home/yourusername/backups backupremote:your_backup_folder
     ```

4. **Test Rclone Configuration**

   ```bash
   rclone ls backupremote:your_backup_folder
   ```

---

### **6. Add Monitoring and Logging Systems**

Monitoring ensures you can track performance, detect issues early, and maintain uptime.

#### **6.1. Deploy Prometheus and Grafana**

These tools work together to collect and visualize metrics.

1. **Create Directory and Docker Compose File**

   ```bash
   mkdir ~/monitoring && cd ~/monitoring
   nano docker-compose.yml
   ```

   **Contents:**

   ```yaml
   version: '3'
   services:
     prometheus:
       image: prom/prometheus:latest
       container_name: prometheus
       volumes:
         - ./prometheus.yml:/etc/prometheus/prometheus.yml
       ports:
         - "9090:9090"
       networks:
         - proxy
       restart: always

     grafana:
       image: grafana/grafana:latest
       container_name: grafana
       environment:
         - GF_SECURITY_ADMIN_PASSWORD=your_grafana_password
       volumes:
         - grafana-data:/var/lib/grafana
       ports:
         - "3002:3000"
       networks:
         - proxy
       restart: always

   volumes:
     grafana-data:

   networks:
     proxy:
       external: true
   ```

2. **Create Prometheus Configuration**

   ```bash
   nano prometheus.yml
   ```

   **Contents:**

   ```yaml
   global:
     scrape_interval: 15s

   scrape_configs:
     - job_name: 'prometheus'
       static_configs:
         - targets: ['localhost:9090']

     - job_name: 'node_exporter'
       static_configs:
         - targets: ['host.docker.internal:9100']
   ```

3. **Deploy Node Exporter for Server Metrics**

   ```bash
   sudo docker run -d --name node_exporter --net=host quay.io/prometheus/node-exporter
   ```

4. **Start Prometheus and Grafana**

   ```bash
   sudo docker-compose up -d
   ```

5. **Configure Reverse Proxy in Nginx Proxy Manager**

   - **Add a New Proxy Host for Grafana**:
     - **Domain Names**: `grafana.yourdomain.com`
     - **Scheme**: `http`
     - **Forward Hostname / IP**: `grafana`
     - **Forward Port**: `3000`
     - **SSL**: Enable and request an SSL certificate via Let's Encrypt.

6. **Access Grafana**

   - Visit `https://grafana.yourdomain.com`
   - **Login Credentials**:
     - **Username**: `admin`
     - **Password**: `your_grafana_password`
   - **Add Prometheus as a Data Source**:
     - Navigate to **Configuration > Data Sources > Add data source**.
     - Select **Prometheus** and set the URL to `http://prometheus:9090`.
   - **Import Dashboards**:
     - Use pre-built dashboards for Node Exporter and other services.

#### **6.2. Deploy Netdata for Real-time Monitoring**

Netdata provides detailed real-time monitoring with minimal resource usage.

1. **Install Netdata**

   ```bash
   bash <(curl -Ss https://my-netdata.io/kickstart.sh)
   ```

2. **Configure Reverse Proxy in Nginx Proxy Manager**

   - **Add a New Proxy Host**:
     - **Domain Names**: `netdata.yourdomain.com`
     - **Scheme**: `http`
     - **Forward Hostname / IP**: `localhost`
     - **Forward Port**: `19999`
     - **SSL**: Enable and request an SSL certificate via Let's Encrypt.

3. **Access Netdata Dashboard**

   - Visit `https://netdata.yourdomain.com` to view real-time server metrics.

#### **6.3. Implement ELK Stack (Optional Advanced Logging)**

For comprehensive logging, the ELK stack (Elasticsearch, Logstash, Kibana) can be deployed. Given the server’s resources, this is feasible but optional based on your portfolio needs.

**Due to complexity and resource considerations, this step is optional and recommended only if you wish to demonstrate advanced logging capabilities.**

---

### **7. Include Additional Projects for Portfolio Enhancement**

Leverage your server’s resources to demonstrate a breadth of IT systems administration skills.

#### **7.1. Infrastructure as Code (IaC) with Ansible**

Automate server configurations and deployments using Ansible playbooks.

1. **Install Ansible on Your Local Machine**

   ```bash
   sudo apt install ansible -y
   ```

2. **Create Ansible Playbooks**

   - **Example: Install Docker and Docker Compose**

     ```yaml
     # install_docker.yml
     - hosts: yourserver
       become: yes
       tasks:
         - name: Update apt repository
           apt:
             update_cache: yes

         - name: Install dependencies
           apt:
             name: ['apt-transport-https', 'ca-certificates', 'curl', 'gnupg', 'lsb-release']
             state: present

         - name: Add Docker’s official GPG key
           apt_key:
             url: https://download.docker.com/linux/ubuntu/gpg
             state: present

         - name: Add Docker repository
           apt_repository:
             repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_lsb.codename }} stable"
             state: present

         - name: Install Docker
           apt:
             name: ['docker-ce', 'docker-ce-cli', 'containerd.io']
             state: present
             update_cache: yes

         - name: Install Docker Compose
           get_url:
             url: "https://github.com/docker/compose/releases/download/2.21.0/docker-compose-{{ ansible_system | lower }}-{{ ansible_architecture }}"
             dest: /usr/local/bin/docker-compose
             mode: '0755'
     ```

3. **Define Hosts in Inventory File**

   ```ini
   # hosts.ini
   [yourserver]
   your_server_ip ansible_user=yourusername ansible_ssh_private_key_file=~/.ssh/id_rsa
   ```

4. **Run the Playbook**

   ```bash
   ansible-playbook -i hosts.ini install_docker.yml
   ```

5. **Documentation**

   - Document your playbooks and automation processes as part of your portfolio.

#### **7.2. Deploy Nextcloud for Personal Cloud Storage**

Nextcloud provides a powerful, self-hosted cloud solution for file storage, collaboration, and more.

1. **Create Directory and Docker Compose File**

   ```bash
   mkdir ~/nextcloud && cd ~/nextcloud
   nano docker-compose.yml
   ```

   **Contents:**

   ```yaml
   version: '3'
   services:
     db:
       image: mariadb:latest
       container_name: nextcloud_db
       environment:
         - MYSQL_ROOT_PASSWORD=strong_db_password
         - MYSQL_DATABASE=nextcloud
         - MYSQL_USER=nextcloud
         - MYSQL_PASSWORD=strong_user_password
       volumes:
         - db:/var/lib/mysql
       restart: always
       networks:
         - proxy

     app:
       image: nextcloud:latest
       container_name: nextcloud_app
       ports:
         - "8081:80"
       environment:
         - MYSQL_PASSWORD=strong_user_password
         - MYSQL_DATABASE=nextcloud
         - MYSQL_USER=nextcloud
         - MYSQL_HOST=db
       volumes:
         - nextcloud:/var/www/html
       depends_on:
         - db
       networks:
         - proxy
       restart: always

   volumes:
     db:
     nextcloud:

   networks:
     proxy:
       external: true
   ```

2. **Start Nextcloud**

   ```bash
   sudo docker-compose up -d
   ```

3. **Configure Reverse Proxy in Nginx Proxy Manager**

   - **Add a New Proxy Host**:
     - **Domain Names**: `cloud.yourdomain.com`
     - **Scheme**: `http`
     - **Forward Hostname / IP**: `nextcloud_app`
     - **Forward Port**: `80`
     - **SSL**: Enable and request an SSL certificate via Let's Encrypt.

4. **DNS Configuration**

   - Point `cloud.yourdomain.com` to your server's IP address.

5. **Access Nextcloud**

   - Visit `https://cloud.yourdomain.com` and complete the setup wizard.

---

#### **7.3. Deploy a Documentation Site with MkDocs**

Showcase your projects and document your processes using a static site generator.

1. **Choose a Directory and Initialize MkDocs**

   ```bash
   mkdir ~/docs && cd ~/docs
   mkdocs new mydocs
   cd mydocs
   ```

2. **Customize Your Documentation**

   - Edit `mkdocs.yml` and the Markdown files in the `docs/` directory to add your content.

3. **Create Docker Compose File**

   ```bash
   nano docker-compose.yml
   ```

   **Contents:**

   ```yaml
   version: '3'
   services:
     mkdocs:
       image: squidfunk/mkdocs-material:latest
       container_name: mkdocs
       ports:
         - "8082:8000"
       volumes:
         - ./mydocs:/docs
       command: mkdocs serve --dev-addr=0.0.0.0:8000
       networks:
         - proxy
       restart: always
   networks:
     proxy:
       external: true
   ```

4. **Start the Documentation Site**

   ```bash
   sudo docker-compose up -d
   ```

5. **Configure Reverse Proxy in Nginx Proxy Manager**

   - **Add a New Proxy Host**:
     - **Domain Names**: `docs.yourdomain.com`
     - **Scheme**: `http`
     - **Forward Hostname / IP**: `mkdocs`
     - **Forward Port**: `8000`
     - **SSL**: Enable and request an SSL certificate via Let's Encrypt.

6. **DNS Configuration**

   - Point `docs.yourdomain.com` to your server's IP address.

7. **Access Documentation Site**

   - Visit `https://docs.yourdomain.com` to view your MkDocs site.

---

#### **7.4. Deploy Keycloak for User Management and Authentication**

Keycloak provides identity and access management, supporting single sign-on (SSO).

1. **Create Directory and Docker Compose File**

   ```bash
   mkdir ~/keycloak && cd ~/keycloak
   nano docker-compose.yml
   ```

   **Contents:**

   ```yaml
   version: '3'
   services:
     keycloak:
       image: quay.io/keycloak/keycloak:latest
       container_name: keycloak
       environment:
         - KEYCLOAK_ADMIN=admin
         - KEYCLOAK_ADMIN_PASSWORD=strong_admin_password
       ports:
         - "8083:8080"
       command: start-dev
       networks:
         - proxy
       restart: always
   networks:
     proxy:
       external: true
   ```

2. **Start Keycloak**

   ```bash
   sudo docker-compose up -d
   ```

3. **Configure Reverse Proxy in Nginx Proxy Manager**

   - **Add a New Proxy Host**:
     - **Domain Names**: `auth.yourdomain.com`
     - **Scheme**: `http`
     - **Forward Hostname / IP**: `keycloak`
     - **Forward Port**: `8080`
     - **SSL**: Enable and request an SSL certificate via Let's Encrypt.

4. **DNS Configuration**

   - Point `auth.yourdomain.com` to your server's IP address.

5. **Access Keycloak**

   - Visit `https://auth.yourdomain.com` and log in using the admin credentials.

6. **Integrate Keycloak with Other Services**

   - Use Keycloak for authentication in your applications to demonstrate SSO and secure user management.

---

### **8. Resource Optimization**

Efficient resource management ensures smooth operation and stability, especially when hosting multiple services.

#### **8.1. Implement Swap Space**

Prevent out-of-memory errors by adding swap space.

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

#### **8.2. Limit Docker Resource Usage**

Control resource allocation for each Docker container to prevent any single service from monopolizing resources.

- **Example: Limit CPU and Memory in Docker Compose**

  ```yaml
  services:
    vaultwarden:
      # ...
      deploy:
        resources:
          limits:
            cpus: '1.0'
            memory: '512M'
          reservations:
            cpus: '0.5'
            memory: '256M'
  ```

  **Note**: The `deploy` key is primarily for Docker Swarm. For standalone Docker, use `mem_limit` and `cpus` parameters.

#### **8.3. Monitor Resource Usage**

Regularly monitor your server’s resource usage to make informed adjustments.

- **Use Nginx Proxy Manager’s Integrated Logging**
- **Leverage Grafana Dashboards** for visual insights
- **Use Netdata for Real-time Monitoring**

---

### **9. Security Enhancements**

Securing your server and services is paramount to protect against unauthorized access and data breaches.

#### **9.1. Configure UFW Firewall Rules**

Ensure only necessary ports are open.

```bash
# Reset UFW to default
sudo ufw reset

# Deny all incoming and allow outgoing
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH
sudo ufw allow ssh

# Allow web traffic
sudo ufw allow http
sudo ufw allow https

# Allow specific ports for services if needed (e.g., Grafana on 3000)
# Example:
# sudo ufw allow 3000

# Enable UFW
sudo ufw enable
```

#### **9.2. Install and Configure Fail2Ban**

Protect against brute-force attacks by monitoring and banning suspicious IPs.

1. **Install Fail2Ban**

   ```bash
   sudo apt install fail2ban -y
   ```

2. **Configure Fail2Ban**

   ```bash
   sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
   sudo nano /etc/fail2ban/jail.local
   ```

   - **Adjust Settings**:
     - Enable jails for SSH and other services (e.g., Nginx)
     - Configure ban time, max retries, etc.

   **Example for SSH:**

   ```ini
   [sshd]
   enabled = true
   port = ssh
   filter = sshd
   logpath = /var/log/auth.log
   maxretry = 5
   bantime = 3600
   ```

3. **Restart Fail2Ban**

   ```bash
   sudo systemctl restart fail2ban
   ```

#### **9.3. Regular System and Software Updates**

Keep the system and all installed software up to date to patch vulnerabilities.

1. **Enable Unattended Upgrades**

   ```bash
   sudo apt install unattended-upgrades -y
   sudo dpkg-reconfigure --priority=low unattended-upgrades
   ```

2. **Manual Update (Optional)**

   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

#### **9.4. Secure Nginx Proxy Manager**

- **Enable HTTP Strict Transport Security (HSTS)**
- **Use Strong SSL/TLS Settings**
- **Regularly Update Nginx Proxy Manager**

**Note**: Nginx Proxy Manager handles SSL certificates via Let's Encrypt, which simplifies certificate renewals.

---

### **10. Additional Projects to Maximize Server Utilization**

Leverage the extra resources to demonstrate advanced skills and diversify your portfolio.

#### **10.1. Deploy a Virtual Private Network (VPN) with WireGuard**

Securely access your server and network resources remotely.

1. **Install WireGuard Using Docker**

   ```bash
   mkdir ~/wireguard && cd ~/wireguard
   nano docker-compose.yml
   ```

   **Contents:**

   ```yaml
   version: '3.8'
   services:
     wireguard:
       image: linuxserver/wireguard
       container_name: wireguard
       cap_add:
         - NET_ADMIN
         - SYS_MODULE
       environment:
         - PUID=1000
         - PGID=1000
         - TZ=Your/Timezone
         # Generate your own keys or follow the container's instructions
       volumes:
         - ./config:/config
         - /lib/modules:/lib/modules
       ports:
         - "51820:51820/udp"
       sysctls:
         - net.ipv4.conf.all.src_valid_mark=1
         - net.ipv6.conf.all.disable_ipv6=0
       networks:
         - proxy
       restart: always
   networks:
     proxy:
       external: true
   ```

2. **Start WireGuard**

   ```bash
   sudo docker-compose up -d
   ```

3. **Configure Reverse Proxy (If Needed)**

   WireGuard typically requires UDP on port 51820, which isn’t directly supported by Nginx Proxy Manager. Ensure port forwarding is correctly set up in your firewall.

4. **Access and Configure VPN**

   - Follow the container's documentation to generate keys and client configurations.
   - Use the WireGuard client on your devices to connect securely.

#### **10.2. Host a Game Server (e.g., Minecraft)**

Showcase your ability to manage game servers, which involves handling real-time data and resource allocation.

1. **Create Directory and Docker Compose File**

   ```bash
   mkdir ~/minecraft && cd ~/minecraft
   nano docker-compose.yml
   ```

   **Contents:**

   ```yaml
   version: '3'
   services:
     minecraft:
       image: itzg/minecraft-server
       container_name: minecraft
       ports:
         - "25565:25565"
       environment:
         EULA: "TRUE"
         MAX_MEMORY: "4G"
       volumes:
         - ./data:/data
       networks:
         - proxy
       restart: always
   networks:
     proxy:
       external: true
   ```

2. **Start Minecraft Server**

   ```bash
   sudo docker-compose up -d
   ```

3. **Configure Reverse Proxy**

   - **Add a New Proxy Host**:
     - **Domain Names**: `mc.yourdomain.com`
     - **Scheme**: `tcp`
     - **Custom Configuration**: Nginx Proxy Manager primarily handles HTTP/HTTPS. For TCP proxying (required for Minecraft), consider using a specialized proxy or configure directly via firewall.

4. **DNS Configuration**

   - Point `mc.yourdomain.com` to your server's IP address with a SRV record pointing to port `25565`.

5. **Access Minecraft Server**

   - Connect using `mc.yourdomain.com` in your Minecraft client.

**Note**: Hosting game servers can be resource-intensive. Allocate appropriate resources and monitor performance.

#### **10.3. Deploy a Media Streaming Server with Jellyfin**

Jellyfin is an open-source media server for streaming your personal media collection.

1. **Create Directory and Docker Compose File**

   ```bash
   mkdir ~/jellyfin && cd ~/jellyfin
   nano docker-compose.yml
   ```

   **Contents:**

   ```yaml
   version: '3'
   services:
     jellyfin:
       image: jellyfin/jellyfin:latest
       container_name: jellyfin
       ports:
         - "8096:8096"
       volumes:
         - ./config:/config
         - ./media:/media
       networks:
         - proxy
       restart: always
   networks:
     proxy:
       external: true
   ```

2. **Start Jellyfin**

   ```bash
   sudo docker-compose up -d
   ```

3. **Configure Reverse Proxy in Nginx Proxy Manager**

   - **Add a New Proxy Host**:
     - **Domain Names**: `media.yourdomain.com`
     - **Scheme**: `http`
     - **Forward Hostname / IP**: `jellyfin`
     - **Forward Port**: `8096`
     - **SSL**: Enable and request an SSL certificate via Let's Encrypt.

4. **DNS Configuration**

   - Point `media.yourdomain.com` to your server's IP address.

5. **Access Jellyfin**

   - Visit `https://media.yourdomain.com` to set up your media library.

---

### **11. Optimize Resource Utilization**

With multiple services running, it's crucial to ensure efficient resource usage.

#### **11.1. Implement Swap Space**

(If not already done in Section 8.1)

```bash
sudo fallocate -l 8G /swapfile  # Increase swap if necessary
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

#### **11.2. Utilize Resource Limits in Docker**

Ensure that no single container consumes excessive resources.

- **Example: Limit CPU and Memory in Docker Compose**

  ```yaml
  services:
    nextcloud:
      # ...
      deploy:
        resources:
          limits:
            cpus: '2.0'
            memory: '2G'
          reservations:
            cpus: '1.0'
            memory: '1G'
  ```

  **Note**: The `deploy` key is designed for Docker Swarm. For standalone Docker, use `mem_limit` and `cpus`.

#### **11.3. Regularly Monitor Resource Usage**

- **Use Grafana Dashboards**
- **Leverage Netdata for Real-time Insights**
- **Set Alerts for High Utilization**

---

### **12. Enhance Security Across All Services**

Implement comprehensive security measures to protect your server and services.

#### **12.1. Enforce Strong Passwords and Keys**

- **Use SSH Keys Instead of Passwords** for all server access.
- **Generate Strong Passwords** for all services, preferably using a password manager like Vault Warden.

#### **12.2. Enable Two-Factor Authentication (2FA)**

- **Enable 2FA** on services like Gitea, Jenkins, and Nextcloud to add an extra layer of security.

#### **12.3. Regular Security Audits**

- **Use Tools like Lynis** to perform security audits.

  ```bash
  sudo apt install lynis -y
  sudo lynis audit system
  ```

- **Review Logs Regularly** for suspicious activities.

#### **12.4. Secure Sensitive Data**

- **Store Secrets Securely** using Docker secrets or environment variables with limited access.
- **Encrypt Data at Rest** where possible, especially for backups.

---

## **Conclusion**

By upgrading to the **Hetzner CX42** plan, you have significantly more resources to deploy a comprehensive suite of services that not only meet your initial requirements but also expand your portfolio with advanced projects. This setup showcases your abilities in server management, automation, security, orchestration, and more, making it an impressive demonstration of your skills as an IT systems administrator.

---

## **Next Steps**

1. **Provision the Hetzner CX42 Server**

   - Deploy **Ubuntu 22.04 LTS** on your Hetzner CX42 server.

2. **Follow the Step-by-Step Implementation**

   - Proceed sequentially, ensuring each component is correctly set up and functioning before moving to the next.

3. **Document Your Setup**

   - Keep detailed records of configurations, challenges, and solutions.
   - Consider creating a blog or documentation site (as outlined) to showcase your work.

4. **Test Each Service Thoroughly**

   - Verify accessibility, performance, and security.
   - Perform stress tests to ensure stability under load.

5. **Optimize and Refine**

   - Continuously monitor performance and make necessary adjustments.
   - Update services regularly to maintain security and functionality.

6. **Expand Your Portfolio**

   - Add new projects as your skills grow.
   - Consider contributions to open-source projects or developing your own tools.

---

## **Additional Considerations**

- **Domain Management**: Ensure your domain registrar allows you to manage DNS records for subdomains effectively.
- **SSL Management**: Nginx Proxy Manager automates Let’s Encrypt SSL certificate renewals, simplifying HTTPS management.
- **Data Privacy and Compliance**: If handling personal data, ensure compliance with relevant data protection regulations.
- **Cost Management**: Monitor any additional costs, such as cloud storage for off-site backups, to stay within your budget.

---

## **Support**

If you encounter any issues or require further assistance during the setup process, feel free to reach out. Building this comprehensive server setup will not only serve your immediate needs but also significantly bolster your IT systems administration portfolio.

Happy deploying!

