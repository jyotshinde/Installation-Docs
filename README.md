## Lab: Install SonarQube on EC2

For the lab, we will keep SonarQube and PostgreSQL on the **same VM**. This is ok for training, and close enough to a small production style deployment on VMs.

### 1. Create the EC2 instance

* AMI: Ubuntu 22.04 LTS
* Instance type: c7i-flex.large (SonarQube likes RAM)
  > **Note:** The instance type **c7i-flex.large** is eligible under the AWS **credits-based free tier**, so you will not incur charges as long as you remain within your credit balance.

* Root volume: 20 GB or more

**Security group (SonarQube VM)**

* Allow SSH (22) from your IP
* Allow HTTP (9000) from your IP and from the Jenkins VM


### 2. Connect to the instance

```bash
ssh -i <your-key>.pem ubuntu@<sonarqube-ec2-public-ip>
```
**Note:** Ensure your SSH private key is readable only by you.
Run:

```bash
chmod 600 ~/.ssh/<your-prvate-key>  
chown $(whoami):$(whoami) ~/.ssh/<your-prvate-key>  
```

**Suggested hostname:** `sonarqube`
This makes the server easy to identify in SSH sessions, monitoring dashboards, and DNS-based service discovery.

```bash
# Set hostname and reload your shell
sudo hostnamectl set-hostname sonarqube
exec bash
```

**Setting the correct timezone**
Set the machine timezone so logs and timestamps are consistent with your locality.
I'll use `Asia/Kolkata`; pick the timezone that matches your location.

```bash
# set timezone to Asia/Kolkata
sudo timedatectl set-timezone Asia/Kolkata

# verify
timedatectl status
# or
date
```

### 3. Install Java 21

SonarQube currently requires Java 21.

```bash
sudo apt-get update
sudo apt-get install -y openjdk-21-jre
java -version
```

You should see Java 21.

### 4. Install PostgreSQL and create DB for SonarQube

In SonarQube, the **database is critical**. It stores:

* All **projects, analyses and measures** (issues, coverage, duplications, ratings, technical debt).
* **Rules, quality profiles, quality gates**, user accounts, tokens and general configuration.

Because this data is so important, in production you will often see enterprises run SonarQube’s database on a **separate, self managed PostgreSQL VM** or use a **managed service such as Amazon RDS for PostgreSQL**, so that performance, backups and high availability can be handled independently from the SonarQube application VM.

For **development and small test setups**, you will sometimes see **H2** being used. H2 is an **embedded, in memory or file based relational database** that is very easy to start but not designed for heavy, multi user production use. It is fine for quick local trials, but for any serious team environment SonarQube recommends **PostgreSQL** instead.


```bash
# Install PostgreSQL and common extensions
sudo apt install -y postgresql postgresql-contrib

# Enable PostgreSQL to start automatically on boot
sudo systemctl enable postgresql

# Start the PostgreSQL service now
sudo systemctl start postgresql
```

**Create SonarQube database and user**

Below are the exact commands to create the database role and the `sonarqube` database, plus a quick connectivity test.
Run the SQL commands as the `postgres` superuser.

```bash
# open the PostgreSQL shell as the postgres superuser
sudo -u postgres psql
```

Inside `psql`, run:

```sql
-- create a login role for SonarQube with a password
CREATE ROLE sonar WITH LOGIN ENCRYPTED PASSWORD 'StrongPasswordHere';
-- Example: CREATE ROLE sonar WITH LOGIN ENCRYPTED PASSWORD 'sonar';

-- create the SonarQube database owned by that role
CREATE DATABASE sonarqube OWNER sonar;

-- grant privileges on the database to the sonar role
GRANT ALL PRIVILEGES ON DATABASE sonarqube TO sonar;

-- exit psql
\q
```

Quick verification (run on the VM shell):

```bash
# list roles to confirm the 'sonar' role exists
sudo -u postgres psql -c "\du"

# list databases and owners to confirm the 'sonarqube' DB exists
sudo -u postgres psql -c "\l"

# test connecting exactly as SonarQube will (replace password if needed)
PGPASSWORD='StrongPasswordHere' psql -U sonar -h localhost -d sonarqube -c "\dt"
```

If the last command shows an empty relation list (no tables) and no authentication error, the database and user are configured correctly.
Update `sonar.jdbc.username`, `sonar.jdbc.password`, and `sonar.jdbc.url` in `/opt/sonarqube-current/conf/sonar.properties` to match these values and restart SonarQube.



### 5. Tune kernel settings required by SonarQube

SonarQube uses Elasticsearch internally and needs some kernel tweaks.

```bash
# Increase the maximum number of memory map areas (required by Elasticsearch)
echo 'vm.max_map_count=524288' | sudo tee -a /etc/sysctl.d/99-sonarqube.conf

# Increase the maximum number of file handles
echo 'fs.file-max=131072' | sudo tee -a /etc/sysctl.d/99-sonarqube.conf

# Apply the new sysctl settings
sudo sysctl --system
```

Set ulimits for the `sonar` user by adding to `/etc/security/limits.d/99-sonarqube.conf`:

```bash
# Allow many open files for the sonar user
echo 'sonar   -   nofile   131072' | sudo tee /etc/security/limits.d/99-sonarqube.conf

# Allow many processes for the sonar user
echo 'sonar   -   nproc    8192'   | sudo tee -a /etc/security/limits.d/99-sonarqube.conf
```

> **Note:** These kernel and ulimit tweaks give Elasticsearch (used by SonarQube) enough memory mappings and file descriptors to index code reliably.
> Apply them before starting SonarQube to avoid startup, indexing, or resource-limit failures.


### 6. Create a dedicated sonar user

```bash
# Create a dedicated 'sonar' user with its home directory at /opt/sonarqube
sudo useradd -m -d /opt/sonarqube -s /bin/bash sonar
```

### 7. Download and install SonarQube

Reference: https://www.sonarsource.com/products/sonarqube/downloads/

Check the LTS download URL from SonarQube site; as an example:

```bash
# switch to a temporary folder for downloads (keeps /tmp clean and avoids permission issues)
cd /tmp

# download the SonarQube community zip, follow redirects (-L) and write output to sonarqube.zip (-o)
curl -L -o sonarqube.zip "https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-25.11.0.114957.zip"

# install unzip non-interactively (-y auto-accepts prompts)
sudo apt-get update
sudo apt-get install -y unzip

# extract the downloaded archive into /tmp (unzip sonarqube.zip)
unzip sonarqube.zip

# move the extracted folder to a stable location for SonarQube deployment
sudo mv sonarqube-25.11.0.114957 /opt/sonarqube-current

# change ownership recursively (-R) so the sonar system user owns all files and directories
sudo chown -R sonar:sonar /opt/sonarqube-current
```


You can keep `/opt/sonarqube-current` as a symlink later when upgrading.

> In enterprise-grade implementations you would typically see the **Data Center Edition (DCE)** used because it provides application-level high availability.
**DCE** enables clustered SonarQube: multiple app nodes behind a load balancer, dedicated Elasticsearch search nodes, and a shared external database for resilience, scale and zero-downtime maintenance.


### 8. Configure SonarQube to use PostgreSQL

We are configuring SonarQube to connect to the PostgreSQL database you created.
**PostgreSQL default port is `5432`** — use it in the JDBC URL unless your DB uses a different port.

```bash
# edit the main SonarQube configuration file as the 'sonar' user
# (use your preferred editor: vim/nano)
sudo -u sonar vim /opt/sonarqube-current/conf/sonar.properties
```

Uncomment and set these lines (replace the password/host if needed):

```properties
# DB username for SonarQube (the DB user you created)
sonar.jdbc.username=sonar

# DB password for that user; keep this secret and match your DB setup
sonar.jdbc.password=StrongPasswordHere

# JDBC URL: jdbc:postgresql://<host>:<port>/<database>
# default PostgreSQL port is 5432 — using localhost because DB is on same VM
sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube
```

Notes and quick checks:

* If PostgreSQL runs on a different host, replace `localhost` with the DB host or IP and ensure the DB accepts remote connections (`listen_addresses` and `pg_hba.conf`).
* After saving, restart SonarQube so it picks up the new DB settings.
* If the SonarQube process cannot connect, check PostgreSQL is running and that the `sonar` user has access to the `sonarqube` database.


### 9. Create a systemd service for SonarQube

**What a systemd unit file is:** a small config that tells the OS how to start, stop, and manage a service, which user runs it, what it depends on, restart policy, and resource limits.

Create the unit file (edit as root):

```bash
sudo vim /etc/systemd/system/sonarqube.service
```

Paste this exact unit file (no inline comments inside the file):

```ini
[Unit]
Description=SonarQube service
After=network.target postgresql.service

[Service]
Type=forking
User=sonar
Group=sonar
ExecStart=/opt/sonarqube-current/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube-current/bin/linux-x86-64/sonar.sh stop
Restart=on-failure
LimitNOFILE=131072
LimitNPROC=8192

[Install]
WantedBy=multi-user.target
```

Reload systemd and start the service:

```bash
# reload systemd to pick up the new unit
sudo systemctl daemon-reload

# enable SonarQube to start on boot
sudo systemctl enable sonarqube

# start SonarQube now
sudo systemctl start sonarqube

# check current status; it should become active (running)
sudo systemctl status sonarqube
```

**Note: unit file directive explanations (keep these out of the unit file)**

* `Description` — human-friendly name for the service.
* `After` — start this service after the listed units are up (network and PostgreSQL here).
* `Type=forking` — the service forks to the background; systemd treats the original process as the starter.
* `User` / `Group` — the system account that runs the service (`sonar`) for least privilege.
* `ExecStart` — full path to the command that starts SonarQube. It must be executable.
* `ExecStop` — full path to the command that stops SonarQube cleanly.
* `Restart=on-failure` — automatically restart the service if it crashes or exits with an error.
* `LimitNOFILE` — raise the open-files limit; Elasticsearch needs many file descriptors.
* `LimitNPROC` — raise the allowed process count for the service user.
* `WantedBy=multi-user.target` — makes the service start during normal system boot.

**Operational tips:**

* If status is not `active (running)`, check live logs: `sudo journalctl -u sonarqube -f`.
* Confirm the `sonar` user owns `/opt/sonarqube-current` and that the `sonar` user has the ulimits set earlier.
* After editing the unit file, always run `sudo systemctl daemon-reload` to apply changes.

**Troubleshooting**
Check the SonarQube logs (look for ERROR / bootstrap failures)

```bash
# show recent sonar logs (general), web and elasticsearch logs
sudo tail -n 200 /opt/sonarqube-current/logs/sonar.log
sudo tail -n 200 /opt/sonarqube-current/logs/web.log
sudo tail -n 200 /opt/sonarqube-current/logs/es.log
```


### 10. Access the SonarQube UI

In your browser:

```text
http://<sonarqube-ec2-public-ip>:9000
```

Default login:

* Username: `admin`
* Password: `admin`
