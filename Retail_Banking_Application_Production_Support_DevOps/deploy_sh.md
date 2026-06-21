# deploy.sh — Every Line Explained (Zero to Understanding)

---

## BEFORE WE READ — what is this script and who calls it?

Remember Stage 6 in the Jenkinsfile?
Jenkins SSH'd into each app server and ran:
`bash /opt/scripts/deploy.sh 42`

THIS is that script. It lives on each of the 3 app servers at the
path `/opt/scripts/deploy.sh`. Jenkins didn't create this file —
YOU (the DevOps engineer) wrote it and placed it on the servers
as part of setting up the deployment process.

**Its one job**: Take the OLD running container off, put the NEW one on.
Like a car mechanic swapping out an old engine for a new one,
while making sure the new one actually starts before declaring success.

**Who calls it**: Jenkins (via SSH from Stage 6 and Stage 8).
**When it runs**: On each of the 3 app servers, during every deployment.
**What it receives**: One input — the image tag number (e.g., 42).

---

## THE COMPLETE SCRIPT — every line explained

```bash
#!/bin/bash
```
**What this is**: Called a "shebang" (pronounced shuh-bang).
It's the very first line of every shell script.
It tells the operating system: "use the program at /bin/bash
to interpret and run the commands in this file."

Without this line, the OS doesn't know HOW to run the file.
It's like telling someone "read this in English" before handing
them a document — sets the language/interpreter.

---

```bash
set -e
```
**What this does**: "If ANY command in this script fails,
stop the entire script immediately. Don't continue."

Without `set -e`:
If `docker pull` fails (network error, image not found),
the script would CONTINUE trying to stop and start a container
that doesn't exist yet → chaos and confusing errors.

With `set -e`:
If `docker pull` fails → script stops right there.
Jenkins sees the script exited with an error.
Jenkins marks the Deploy stage as FAILED.
Team is alerted. Nobody gets confused by what happened next.

**Interview line**: "`set -e` is a safety mechanism — it ensures
that if any step in the deployment fails, we stop immediately
rather than blindly continuing with subsequent steps that depend
on the failed step succeeding."

---

```bash
IMAGE_TAG=$1
```
**What this means**: "Take the first thing passed to this script
and store it in a variable called IMAGE_TAG."

When Jenkins called: `bash /opt/scripts/deploy.sh 42`
The `42` is the "first argument" → `$1` captures it → `IMAGE_TAG = 42`

**Why pass it as a variable instead of hardcoding?**
If it were hardcoded (`IMAGE_TAG=42`), every time you deployed
a new version, someone would have to SSH into all 3 servers
and manually change this number in the script. Error-prone.
By receiving it as an argument from Jenkins, the script is
REUSABLE for any version. Jenkins passes the correct number
automatically every time.

---

```bash
REPO="yourteam/banking-app"
CONTAINER_NAME="banking-app"
HEALTH_CHECK_URL="http://localhost:8080/health"
```
**What these are**: More variables — constants that don't change
between deployments.

`REPO` = the Docker Hub location of our images.
Every image we built lives under this "folder" on Docker Hub.
`yourteam` = the organization/account name on Docker Hub.

`CONTAINER_NAME` = the name we give to our running container.
Why have a consistent name? So we can always refer to it by the
same name when stopping/starting, regardless of which version is
running. Like naming your car "my car" — you always know which
one you mean.

`HEALTH_CHECK_URL` = the URL Jenkins will check AFTER starting
the container to confirm the app is alive.
`localhost` = "this same server" (we're checking the app
that just started on THIS machine).
Port 8080 = the port Tomcat/our app listens on.
`/health` = a special endpoint our banking app provides —
when you call this URL, the app responds "I'm alive" (HTTP 200).

---

```bash
echo "=========================================="
echo "Starting deployment — version: $IMAGE_TAG"
echo "=========================================="
```
**What 'echo' does**: Prints text to the terminal/log.
These lines produce visible output in the Jenkins console log.

**Why does this matter?**
When you're troubleshooting a failed deployment at 2am,
you open the Jenkins console log and the first thing you see is:
"Starting deployment — version: 42"
Immediately you know: which version was being deployed when it failed.
Without this, you'd have to go hunting for that information elsewhere.

Good logging = faster debugging. Always echo important information
at the start of a script.

---

```bash
echo "Pulling image: $REPO:$IMAGE_TAG"
docker pull $REPO:$IMAGE_TAG
```
**What 'docker pull' does**:
Downloads the Docker image from Docker Hub to THIS server.

Full command expands to: `docker pull yourteam/banking-app:42`

**Behind the scenes**:
1. Docker contacts Docker Hub over the internet/network
2. Finds the image "yourteam/banking-app" with tag "42"
3. Downloads each LAYER of the image
4. If a layer already exists from a previous pull (unchanged layers),
   Docker skips it — only downloads what's new
5. Stores the complete image locally on this server
6. Now the server has "yourteam/banking-app:42" ready to run

**What if this fails?**
- Image doesn't exist on Docker Hub (Stage 5 push failed earlier?)
- Network issue between server and Docker Hub
- Docker Hub is down
- Authentication issue (if private registry)
`set -e` catches the failure → script stops → Jenkins alerts team.

---

```bash
echo "Stopping old container..."
docker stop $CONTAINER_NAME || true
docker rm $CONTAINER_NAME || true
```
**What 'docker stop' does**:
Sends a SIGTERM signal to the running container.
This tells the application INSIDE: "please shut down gracefully."
The app gets a chance to:
- Finish processing current requests
- Save any pending data
- Release database connections properly
- Then exit cleanly

After a grace period (default 10 seconds), if the container
still hasn't stopped, Docker sends SIGKILL (force kill).

**What 'docker rm' does**:
Removes (deletes) the stopped container.
The container is gone — but the IMAGE still exists.
Think of it like: the container is a running instance of the image.
You're removing the instance, not the blueprint.

**What does '|| true' mean?**
`||` = "OR"
`|| true` = "OR if the previous command fails, that's okay, continue."

Why needed?
If this is the VERY FIRST deployment on a fresh server,
there is no old container to stop or remove.
`docker stop banking-app` would fail with "container not found."
Without `|| true`, `set -e` would stop the script here.
But this isn't a real error — there was just nothing to stop.
`|| true` says: "if nothing to stop/remove, that's fine, keep going."

**The sequence**: stop THEN remove. Why not just remove directly?
A running container cannot be removed — Docker won't allow it.
You must stop it first, then remove it. Order matters.

---

```bash
echo "Starting new container..."
docker run -d \
    --name $CONTAINER_NAME \
    -p 8080:8080 \
    --restart always \
    -e DB_HOST="bankingdb.rds.amazonaws.com" \
    -e DB_PORT="1433" \
    -e DB_NAME="bankingdb" \
    -e DB_USER="appuser" \
    -e DB_PASS="$DB_PASSWORD" \
    $REPO:$IMAGE_TAG
```
**This is the most important command** — this actually STARTS the new
version of the application. Let's go through every flag:

`docker run` = "create and start a new container from this image"

`-d` = detached mode.
Run the container in the BACKGROUND.
Without -d, the terminal would be "captured" by the running container
and you'd have to press Ctrl+C to stop it.
With -d, the container runs in the background and the script
continues to the next line immediately.

`--name $CONTAINER_NAME` = "name this container 'banking-app'."
Consistent name so we can always refer to it by name.
Next deployment: `docker stop banking-app` — always works.

`-p 8080:8080` = port mapping. Format is HOST_PORT:CONTAINER_PORT.
Left 8080 = the server's (EC2's) port that listens for traffic.
Right 8080 = the port INSIDE the container where the app runs.
This "connects" the outside world to the app inside the container.

**Without -p**: the app runs inside the container but nobody
from outside (including the ALB) can reach it.
The container is a sealed box — -p is like cutting a hole in the
box with a specific size to let traffic through.

`--restart always` = "if this container ever stops unexpectedly,
restart it automatically."
If the server reboots → container starts again automatically.
If the app crashes inside → Docker restarts the container.
This is your self-healing mechanism — you don't need to manually
restart containers after server reboots or app crashes.

`-e DB_HOST="bankingdb.rds.amazonaws.com"` = environment variable.
Passes the database's network address TO the app running inside.
The app reads this variable on startup to know WHERE the database is.

`-e DB_PORT="1433"` = SQL Server's port number.
1433 is the standard default port for SQL Server.

`-e DB_NAME="bankingdb"` = which database to connect to.
RDS can host multiple databases — this specifies ours.

`-e DB_USER="appuser"` = username for database login.
We create a dedicated application user in SQL Server (not the admin).
Principle of least privilege — app user has only the permissions
it needs (read/write to specific tables), nothing more.

`-e DB_PASS="$DB_PASSWORD"` = database password.
`$DB_PASSWORD` is fetched from environment or Jenkins secrets.
NOT hardcoded here for the same security reason as Docker Hub
credentials — never put passwords in plain text in scripts.

**Why pass database config as environment variables (-e) and not
bake it into the Docker image?**
Critical concept — different environments (QA, Production) have
DIFFERENT databases. If the DB config were inside the image,
you'd need a different image for QA and Production.
With -e, the SAME image runs everywhere — you just pass different
values to the -e flags. One image, multiple environments.
This is called "12-factor app" principle — externalize configuration.

`$REPO:$IMAGE_TAG` = "use this specific image to create the container."
Expands to: `yourteam/banking-app:42`
This is the image we just pulled in the docker pull step.

---

```bash
echo "Waiting 15 seconds for app to start..."
sleep 15
```
**What 'sleep 15' does**: Pauses the script for 15 seconds.
Does nothing for 15 seconds, then continues.

**Why wait?**
When `docker run` executes, the container STARTS — but the
APPLICATION INSIDE takes time to fully initialize.
For a Java/Tomcat application:
- JVM (Java Virtual Machine) starts up
- Tomcat web server initializes
- Application loads, reads config, connects to database
- Only then: ready to serve requests

This process takes 10-30 seconds typically.
If we ran the health check immediately after docker run,
the app would still be starting and the health check would fail
(not because the app is broken, but because it's not ready yet).
15 seconds = reasonable time to let it initialize.

**In production systems**, instead of a fixed sleep, teams use
a retry loop that keeps checking until the app responds or a
timeout is reached. But 15 seconds works reliably for our app.

---

```bash
echo "Running health check..."
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" $HEALTH_CHECK_URL)
```
**What 'curl' does**: Makes an HTTP request to a URL.
Like your browser visiting a webpage, but from the terminal.

`-s` = silent mode. Don't show progress bars or error messages.
We only want the result, not the noise.

`-o /dev/null` = throw away the response BODY.
We don't care what the page says — we only care about the
HTTP status code (200, 404, 500 etc.).
`/dev/null` = a special Linux "trash bin" — anything sent there disappears.

`-w "%{http_code}"` = "print the HTTP status code after the request."
This is what we actually capture.

`HTTP_CODE=$( )` = run the command inside the brackets
and store its output in the variable HTTP_CODE.

**Result**: HTTP_CODE will contain something like:
- `200` = app is running and healthy ✅
- `404` = URL not found (wrong health check path configured)
- `500` = app is running but has an internal error
- `000` = connection refused — app didn't start at all

---

```bash
if [ "$HTTP_CODE" -eq 200 ]; then
    echo "✅ Health check PASSED — app is running on version $IMAGE_TAG"
    exit 0
else
    echo "❌ Health check FAILED — app returned HTTP $HTTP_CODE"
    exit 1
fi
```
**What 'if' does**: Makes a decision based on a condition.
`if [ "$HTTP_CODE" -eq 200 ]` = "if HTTP_CODE equals 200..."
`-eq` = "equal to" (for numbers)

**Two outcomes**:

`exit 0` = "script finished successfully."
In Linux, exit code 0 = success (counterintuitive but standard).
Jenkins sees exit 0 → marks Deploy stage as PASSED (green) ✅
Pipeline continues.

`exit 1` = "script finished with an error."
Any non-zero exit code = failure in Linux.
Jenkins sees exit 1 → marks Deploy stage as FAILED (red) ❌
`set -e` was set at the top, so nothing runs after this.
Jenkins stops the pipeline and notifies the team.

**Why is this health check the most important part of deploy.sh?**
Without it, Jenkins would report "deployment succeeded" even if
the new container started but the app inside immediately crashed.
The container would be running (Docker says "running") but
customers would get errors because the APP inside isn't working.

With the health check, Jenkins only reports success when the app
is ACTUALLY responding to requests. A real end-to-end verification,
not just "did Docker start the container?"

**What happens to customers during the 15-second wait + health check?**
Remember we have 3 servers. When we stop the container on Server 1:
- ALB detects Server 1's health check failing
- ALB routes ALL traffic to Server 2 and Server 3
- Customers continue being served — zero downtime
- Server 1 comes back up with new version → ALB includes it again
- Then Server 2 gets updated, etc.

---

## THE COMPLETE FLOW — what happens on ONE server during deployment

```
Jenkins SSH's in and calls: bash /opt/scripts/deploy.sh 42
                    ↓
set -e activated: any failure = script stops
                    ↓
IMAGE_TAG = 42 (received from Jenkins)
                    ↓
docker pull yourteam/banking-app:42
(downloads new image from Docker Hub — 30-60 seconds)
                    ↓
docker stop banking-app
(sends graceful shutdown signal to old container — up to 10 seconds)
                    ↓
docker rm banking-app
(removes old container — the IMAGE still exists, just the instance is gone)
                    ↓
docker run ... yourteam/banking-app:42
(creates and starts NEW container from new image)
                    ↓
sleep 15
(wait for Java/Tomcat to fully initialize inside the container)
                    ↓
curl http://localhost:8080/health
(check: is the app actually responding?)
                    ↓
HTTP 200? → exit 0 → Jenkins: Deploy PASSED ✅
HTTP other? → exit 1 → Jenkins: Deploy FAILED ❌ → team alerted
```

---

## KEY INTERVIEW POINTS FROM THIS SCRIPT

- `set -e` = fail fast, don't blindly continue on errors
- `$1` = receives the image tag from Jenkins — makes script reusable
- `-p 8080:8080` = how traffic gets into the container from outside
- `--restart always` = self-healing — containers restart automatically
- `-e` flags = environment variables — same image works in QA and Prod
- `|| true` = handling "first deployment" gracefully without errors
- `sleep 15` = Java apps need initialization time before health check
- `exit 0 / exit 1` = how Jenkins knows if deployment succeeded or failed
- Health check = the REAL success criteria, not just "container started"
- Rolling deployment (1 server at a time) = zero customer downtime
