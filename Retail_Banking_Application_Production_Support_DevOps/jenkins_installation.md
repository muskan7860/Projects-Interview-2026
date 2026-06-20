# Jenkins Installation & Setup — Complete Steps (Ubuntu)

## Why Jenkins needs Java first
Jenkins is itself a Java application — it runs inside a Java runtime (JVM).
So Java must be installed before Jenkins can run. Same concept as your banking
app needing Java inside the Docker container.

---

## Step 1 — Update the machine
```bash
sudo apt update && sudo apt upgrade -y
```

---

## Step 2 — Install Java (Jenkins needs Java 17)
```bash
sudo apt install -y openjdk-17-jre
java -version
```
Expected output: `openjdk version "17.x.x"`

---

## Step 3 — Add Jenkins repository and install Jenkins
```bash
# Add Jenkins GPG key (so your machine trusts the Jenkins package)
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

# Add Jenkins to apt sources list
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update and install
sudo apt update
sudo apt install -y jenkins
```

---

## Step 4 — Start Jenkins and enable it on boot
```bash
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins
```
Expected: `Active: active (running)`

---

## Step 5 — Open Jenkins in browser
```
http://localhost:8080
```

---

## Step 6 — Unlock Jenkins (first time only)
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Copy this password → paste into the browser unlock screen.

---

## Step 7 — Browser setup (one time only)
1. Paste the initial admin password
2. Click "Install suggested plugins" — wait for plugins to install
3. Create your admin username and password
4. Click "Save and Finish" → "Start using Jenkins"

---

## Step 8 — Install Docker (Jenkins will need it to build images)
```bash
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
docker --version
```
Expected: `Docker version 24.x.x`

---

## Step 9 — Give Jenkins permission to run Docker commands
By default Jenkins runs as its own user and can't access Docker.
Fix this by adding Jenkins user to the Docker group:
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```
Why this matters: when Jenkins tries to run `docker build` inside a pipeline,
it runs as the "jenkins" user — if that user doesn't have Docker permission,
the build fails with "permission denied" error. This is one of the most
common Jenkins+Docker problems beginners hit.

---

## Step 10 — Verify everything is connected
```bash
sudo systemctl status jenkins
sudo systemctl status docker
```
Both should show: `Active: active (running)`

---

## What to say in interview if asked "how did you set up Jenkins?"
"Jenkins was already set up when I joined, but I've personally installed
and configured it in my own lab environment. The process involves installing
Java first since Jenkins runs on the JVM, then adding the Jenkins apt
repository, installing Jenkins, and starting it as a service. For our
Docker-based pipeline, one important step is adding the Jenkins user to
the Docker group so the pipeline can run Docker commands without permission
errors — that's a common issue teams run into."

---

## Common interview questions on Jenkins setup

**Q: Why do you need to add Jenkins to the Docker group?**
A: Jenkins runs as its own system user called "jenkins." Docker by default
only allows users in the "docker" group to run Docker commands. If you
don't add "jenkins" to the "docker" group, every `docker build` or
`docker run` command in your pipeline will fail with a permission denied error.

**Q: What port does Jenkins run on by default?**
A: Port 8080.

**Q: How do you make Jenkins start automatically after a server reboot?**
A: `sudo systemctl enable jenkins` — this registers Jenkins as a service
that starts automatically on boot.

**Q: Where does Jenkins store its data/config?**
A: `/var/lib/jenkins/` — this is where job configs, build logs, workspace
folders, and plugins are stored. This directory filling up is a very
common cause of Jenkins pipeline failures (disk full issue).
