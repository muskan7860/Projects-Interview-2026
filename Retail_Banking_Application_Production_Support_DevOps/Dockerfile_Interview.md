# Dockerfile — Complete Guide (Format + RUN/CMD/ENTRYPOINT + All Interview Q&A)

---

## PART 1: THE FORMAT OF A DOCKERFILE

A Dockerfile is a plain text file named exactly "Dockerfile" (capital D, no extension).
It lives in the ROOT of your GitHub repository, next to your application code.

The format is simple:
INSTRUCTION  argument(s)

Each line = one instruction.
Docker reads top to bottom, one line at a time.
Each instruction creates one LAYER in the final image.

The available instructions (you only need to know these for interviews):

| Instruction | Purpose |
|---|---|
| FROM | Start from a base image (always first line) |
| RUN | Run a command during BUILD time |
| COPY | Copy files from outside into the image |
| ADD | Like COPY but can also unzip files (rarely used) |
| EXPOSE | Document which port the app uses |
| ENV | Set environment variables inside the image |
| WORKDIR | Set the working directory for subsequent commands |
| CMD | Default command when container STARTS |
| ENTRYPOINT | Mandatory command when container STARTS |

Our Dockerfile only uses: FROM, RUN, COPY, EXPOSE, CMD
That is enough for a real production Java web application.

---

## PART 2: RUN vs CMD vs ENTRYPOINT — CRYSTAL CLEAR

### THE HOUSE ANALOGY (remember this, it explains everything)

Imagine building a house and living in it. Two completely separate phases:
- Phase 1: BUILDING the house (construction — happens ONCE)
- Phase 2: LIVING in the house (daily life — happens EVERY TIME)

---

### RUN = Phase 1 (Building the house — construction time)

Construction workers build your house ONCE.
They install electricity. They paint walls. They fit the kitchen.
These happen during construction and are permanently part of the house.
Nobody does them again every morning.

RUN in Dockerfile = construction worker action.
Runs ONCE when you build the image (docker build).
Result is permanently baked into the image as a layer.
NEVER runs again when containers start.

Example from our Dockerfile:
```dockerfile
RUN rm -rf /usr/local/tomcat/webapps/ROOT
```
"During construction of this image, delete the default Tomcat page."
Happens once at build time.
Every container from this image has it already deleted.
Nobody deletes it again at container startup.

More examples of RUN (what you'd typically see):
```dockerfile
RUN apt-get update && apt-get install -y curl    # install software
RUN mkdir -p /opt/app/logs                       # create a folder
RUN chmod +x /opt/scripts/deploy.sh              # make file executable
```
All of these are BUILD TIME actions — baked into the image permanently.

---

### CMD = Phase 2 (Your daily morning routine — every startup)

After construction, every morning you wake up and make coffee.
That's your default routine. But if a guest stays, THEY can
choose to make tea instead — they can OVERRIDE your routine.

CMD = default command that runs EVERY TIME a container starts.
CAN be overridden — if someone runs docker run myimage bash,
CMD is ignored and bash runs instead of your default command.

Example from our Dockerfile:
```dockerfile
CMD ["catalina.sh", "run"]
```
"Every time a container starts from this image, start Tomcat."
This is the default. Someone can override it if needed.
In practice, nobody overrides it — Tomcat always starts.

The square bracket format ["catalina.sh", "run"] is called EXEC format.
This is the preferred, correct way to write CMD.
The alternative (shell format: CMD catalina.sh run) works but
doesn't handle signals properly — important for graceful shutdown.
Always use exec format (square brackets) in real Dockerfiles.

---

### ENTRYPOINT = The hotel rule that cannot be broken

Hotel rule: "ALL guests MUST show ID at reception. No exceptions."
No guest can skip this. It always happens first, no matter what.

ENTRYPOINT = command that ALWAYS runs. Cannot be overridden.
Even if someone tries to pass a different command to docker run,
ENTRYPOINT runs first and cannot be replaced.

Example (we did NOT use this in our Dockerfile, but know it):
```dockerfile
ENTRYPOINT ["java", "-jar", "banking-app.jar"]
```
"No matter what, always start with java -jar banking-app.jar."

ENTRYPOINT + CMD together (common pattern):
```dockerfile
ENTRYPOINT ["java", "-jar"]
CMD ["banking-app.jar"]
```
ENTRYPOINT = fixed part (always java -jar)
CMD = default argument (banking-app.jar, but can be overridden)
Combined: runs "java -jar banking-app.jar" by default
Someone can override just the CMD part: docker run myimage other-app.jar
Result: "java -jar other-app.jar" — ENTRYPOINT stays, CMD replaced

---

### THE SIMPLEST COMPARISON (memorize this for interviews)

```
RUN         → BUILD TIME    → baked into image   → CAN'T run at startup
CMD         → STARTUP TIME  → runs every start   → CAN be overridden
ENTRYPOINT  → STARTUP TIME  → runs every start   → CANNOT be overridden
```

In our Dockerfile:
- RUN  = delete default Tomcat page (build time, done once, permanent)
- CMD  = start Tomcat (every container startup, default, could be overridden)
- ENTRYPOINT = NOT USED (CMD is sufficient for our use case)

---

## PART 3: OUR COMPLETE DOCKERFILE WITH FULL EXPLANATION

```dockerfile
FROM tomcat:9.0-jre17
RUN rm -rf /usr/local/tomcat/webapps/ROOT
COPY target/banking-app.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

### Line 1: FROM tomcat:9.0-jre17
EASY: "Start with a pre-built box that already has Java and Tomcat inside."
Instead of installing Java and Tomcat ourselves (20+ lines),
we use an official image where it's already done correctly.
tomcat = base image name on Docker Hub
9.0 = Tomcat version (pinned — never use :latest)
jre17 = Java 17 Runtime included
Creates: Layer 1 (foundation — cached after first build)

### Line 2: RUN rm -rf /usr/local/tomcat/webapps/ROOT
EASY: "Delete Tomcat's default welcome page."
Fresh Tomcat comes with an Apache welcome page in ROOT folder.
If we don't delete it, it conflicts with our banking app.
RUN = happens at BUILD time, once, baked into image permanently.
Creates: Layer 2 (cached — this line never changes)

### Line 3: COPY target/banking-app.war /usr/local/tomcat/webapps/ROOT.war
EASY: "Take our application (WAR file) and put it inside the image."
target/banking-app.war = on Jenkins server (Maven created this in Stage 2)
/usr/local/tomcat/webapps/ROOT.war = destination inside the image
Named ROOT.war so app is served at http://server:8080/ (not /banking-app/)
Tomcat auto-deploys any .war file placed in webapps/ — no extra config.
Creates: Layer 3 (CHANGES every build — new code = new WAR file)

### Line 4: EXPOSE 8080
EASY: "Write on the blueprint that this container uses port 8080."
Does NOT actually open the port — that happens in deploy.sh with -p 8080:8080
It's documentation for humans and container tools like Kubernetes.
Creates: Layer 4 (cached — never changes)

### Line 5: CMD ["catalina.sh", "run"]
EASY: "When a container starts, automatically start Tomcat."
catalina.sh = Tomcat's startup script (pre-installed in base image)
run = start Tomcat in FOREGROUND (keeps container alive)
If background: container would die immediately after CMD finishes.
Foreground: container stays alive as long as Tomcat runs.
Creates: Layer 5 (cached — never changes)

---

## PART 4: HOW TO ANSWER "WALK ME THROUGH YOUR DOCKERFILE"

Say this (practice out loud until natural):

"Our Dockerfile is 5 lines — minimal and clean.

We start with FROM tomcat:9.0-jre17 — an official base image
that has Java 17 and Tomcat 9.0 pre-installed. We pin the exact
version rather than using 'latest' to ensure our builds are
always reproducible — same base every time.

Next, we RUN a command to delete Tomcat's default ROOT webapp.
This is a build-time action — it runs once when we build the image
and is permanently baked in. Without this, Tomcat's default welcome
page would conflict with our banking application.

Then we COPY our WAR file — which Maven built in the previous
pipeline stage — into Tomcat's webapps directory, naming it ROOT.war.
Tomcat automatically detects and deploys any WAR file placed there
on startup, so no additional deployment configuration is needed.
We name it ROOT.war specifically so our app is served at the root
URL without any sub-path.

We EXPOSE port 8080 to document that the application listens there —
the actual port binding to the host happens in our deployment script.

Finally, CMD starts Tomcat in the foreground using catalina.sh run.
Foreground is important — the container stays alive as long as
Tomcat runs, and Tomcat automatically picks up and deploys our
ROOT.war on startup.

The whole thing is 5 lines. Simple, clean, production-ready."

---

## PART 5: ALL DOCKERFILE INTERVIEW QUESTIONS AND ANSWERS

### BASIC QUESTIONS

Q1: What is a Dockerfile?
A: A Dockerfile is a text file containing instructions for building
a Docker image. Docker reads it top to bottom, executes each
instruction, and creates a layered image. Each instruction becomes
one layer. The final image is a complete, portable package containing
the application and everything it needs to run.

---

Q2: What is the difference between an image and a container?
A: An image is a read-only template — the blueprint.
A container is a running instance created from that image.
The relationship: image is like a recipe, container is the
actual cake made from that recipe.
You can create many containers from the same image simultaneously —
like baking many cakes from one recipe.

---

Q3: What is the difference between RUN, CMD, and ENTRYPOINT?
A: RUN executes during image BUILD time — it runs once when you
build the image and the result is baked into the image as a layer.
We use it for things like installing software or deleting files.

CMD executes at CONTAINER STARTUP time — every time a container
is created from the image. It defines the default startup command
and can be overridden when running the container.

ENTRYPOINT also runs at container startup but cannot be overridden.
It always runs regardless of what command is passed to docker run.

In our Dockerfile: RUN deletes the Tomcat default page at build time.
CMD starts Tomcat at container startup. We don't use ENTRYPOINT
because CMD is sufficient — we always want Tomcat to start
and don't need to prevent overriding.

---

Q4: Why do you pin the base image version (9.0-jre17) instead of using 'latest'?
A: 'latest' is a moving target — it changes whenever a new version
is released. If we used FROM tomcat:latest and someone built
the image 6 months later, they might get Tomcat 10 instead of 9.
Our application might not be compatible with Tomcat 10.
By pinning to 9.0-jre17, we get the exact same base image
every single time — builds are reproducible and predictable.
This is especially important in production where unexpected
upgrades can cause failures.

---

Q5: What is Docker layer caching and why does it matter?
A: Each instruction in a Dockerfile creates a layer. Docker caches
each layer after it's built. On the next build, if a layer's
instruction hasn't changed, Docker reuses the cached version
instead of rebuilding it.

In our Dockerfile, FROM and RUN layers never change — they're
cached after the first build and reused in every subsequent build.
Only the COPY layer changes (new WAR file each build).

Result: First build takes 3 minutes (all layers built from scratch).
Every build after takes 30 seconds (only COPY layer rebuilds).
In a CI/CD pipeline that runs multiple times per day, this
saves significant time.

---

Q6: Why do we put COPY after RUN in the Dockerfile? Does order matter?
A: Yes, order matters significantly for caching efficiency.
Layers that don't change should come FIRST, layers that change
frequently should come LAST.

In our Dockerfile:
- FROM and RUN never change → first → always cached
- COPY changes every build → last → always rebuilds

If we put COPY before RUN:
Every time the WAR file changes (every build), Docker would
invalidate the cache from COPY onwards — meaning RUN would
also rebuild unnecessarily even though it didn't change.

Rule: stable instructions first, frequently-changing last.

---

Q7: What is a .dockerignore file and did you use it?
A: .dockerignore works like .gitignore — it tells Docker which
files to EXCLUDE when building the image. Without it, Docker
sends the entire project folder to the build context, including
things like test files, local config, node_modules, or .git folder.
These are unnecessary in the image and slow down the build.

In our project we used .dockerignore to exclude:
- src/test/ (test files — not needed in production image)
- .git/ (version control history — not needed)
- *.md (documentation — not needed)
- local config files

This reduces build context size and speeds up docker build.

---

### INTERMEDIATE QUESTIONS

Q8: How do you reduce Docker image size? (very commonly asked)
A: Several techniques we used:

1. Use a minimal base image — instead of a full Ubuntu image
   with everything installed, use a specific image (tomcat:9.0-jre17)
   that has only what we need. Even better in some cases: Alpine-based
   images (tomcat:9.0-jre17-alpine) which are much smaller.

2. Combine RUN commands — each RUN creates a layer.
   Instead of:
   RUN apt-get update
   RUN apt-get install -y curl
   RUN apt-get clean
   
   Use one RUN:
   RUN apt-get update && apt-get install -y curl && apt-get clean
   
   This creates ONE layer instead of three.

3. Clean up in the same RUN that installs — if you install
   something and clean up in separate RUN commands, the cache/temp
   files are still captured in the install layer.
   Clean in the same command to avoid this.

4. Use .dockerignore — exclude unnecessary files from build context.

5. Remove unnecessary files — delete package manager caches,
   temp files, documentation that isn't needed at runtime.

Interview line: "We used a specific Tomcat base image rather than
a full OS image, and we kept our Dockerfile minimal — only 5 instructions.
We also used .dockerignore to exclude test files and documentation
from the build context."

---

Q9: What is a multi-stage build? (senior-level question — know what it is)
A: Multi-stage builds allow you to use multiple FROM instructions
in one Dockerfile. Each FROM starts a new stage. You can copy
artifacts FROM one stage to another, leaving behind everything
that was only needed for building.

Example for Java app:
Stage 1 (build stage): Use full Maven + JDK image to compile code
Stage 2 (runtime stage): Use minimal JRE image, copy ONLY the JAR/WAR

Result: Final image only contains JRE + the app.
No Maven, no JDK, no source code, no build tools.
Can reduce image size from 600MB to 100MB.

In our project we didn't implement multi-stage builds during my time
there, but I'm aware of the pattern and we discussed it as
a future optimization.

---

Q10: What is Docker Compose and why would you use it?
A: Docker Compose is a tool for defining and running
MULTI-CONTAINER applications using a single YAML file (docker-compose.yml).

Without Compose, to run our banking app locally for testing you'd:
1. Run one command to start the SQL Server container
2. Run another command to start the banking app container
3. Manually configure them to find each other
4. Repeat every time you restart

With Compose, you define BOTH containers in one file:
```yaml
version: '3'
services:
  app:
    image: banking-app:latest
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=database
    depends_on:
      - database
  database:
    image: mcr.microsoft.com/mssql/server:2019-latest
    environment:
      - SA_PASSWORD=YourPassword123
      - ACCEPT_EULA=Y
```

One command: docker-compose up
Both containers start, in the right order, connected to each other.

We used Docker Compose for LOCAL DEVELOPMENT AND TESTING only —
developers could spin up the full app+database stack on their
laptops with one command. In production, we deployed individual
containers via our Jenkins pipeline and deploy.sh, not Compose.

---

Q11: What is the difference between COPY and ADD in Dockerfile?
A: Both copy files into the image, but ADD has extra features:
- ADD can automatically unzip/extract .tar.gz files
- ADD can download files from a URL

For simply copying local files, COPY is preferred because:
- It's explicit — does exactly one thing (copy files)
- ADD's extra features can cause unexpected behaviour
- COPY makes the Dockerfile's intent clearer

Rule of thumb: always use COPY unless you specifically need ADD's
extra capabilities (auto-extraction or URL download).
We used COPY in our Dockerfile for this reason.

---

### TROUBLESHOOTING QUESTIONS

Q12: Docker build is failing — how do you troubleshoot it?
A: First I check the exact error message from the build output —
docker build shows which instruction failed and why.

Common causes:
1. COPY fails — source file doesn't exist.
   Check if Maven build ran successfully first (is target/banking-app.war there?)
   In our pipeline: Stage 2 (Build) must complete before Stage 4 (Docker Build)
   
2. RUN fails — command not found or permission denied.
   Check if the base image has the required tools available.
   
3. FROM fails — base image can't be pulled.
   Check internet/network connectivity from build server.
   Check if Docker Hub is accessible.
   Check if the image tag actually exists on Docker Hub.
   
4. Build context too large — build takes very long to start.
   Add a .dockerignore file to exclude unnecessary files.
   
5. Disk full during build.
   Docker build uses temporary space.
   Run: docker system prune -f to clean unused images/cache.
   Check: df -h on Jenkins server to verify disk space.

---

Q13: Container starts but application is not accessible on port 8080
A: This is a port mapping issue. I check in this order:

1. Is the container actually running?
   docker ps — check if container shows as "Up"
   
2. Is the port mapped correctly?
   docker ps shows port mapping — should show 0.0.0.0:8080->8080/tcp
   If no port mapping: re-run with -p 8080:8080
   
3. Is the app inside the container actually running?
   docker exec -it <container-name> ps aux
   Look for java/catalina process
   
4. Is there an error inside the container?
   docker logs <container-name>
   Look for startup errors from Tomcat or the app
   
5. Is EXPOSE in the Dockerfile? (not the real fix but check)
   EXPOSE doesn't open ports — -p flag does.
   But check if -p was forgotten in docker run.

---

Q14: Container stops immediately after starting — what do you check?
A: This almost always means the main process (CMD) exited.
I check:

1. docker logs <container-name> — see what error occurred before exit
2. docker ps -a — shows stopped containers and their exit code
   Exit code 1 = application error
   Exit code 137 = killed by OS (usually out of memory — OOM)
   Exit code 139 = segmentation fault

Most common cause in our setup:
CMD is running Tomcat in BACKGROUND instead of FOREGROUND.
If CMD uses & or runs as daemon, the CMD completes immediately
and Docker kills the container.
Fix: ensure CMD ["catalina.sh", "run"] — "run" = foreground mode.

Second common cause:
Application fails to start due to wrong config (wrong DB host/port
in environment variables). Check docker logs for connection errors.

---

Q15: Docker image size is too large — what do you do?
A: I approach this in order of impact:

1. Check current size: docker images — see the SIZE column
2. Use smaller base image: tomcat:9.0-jre17-alpine is much smaller
   than tomcat:9.0-jre17 (Alpine Linux = 5MB vs Debian = 120MB)
3. Combine RUN commands to reduce layers
4. Add .dockerignore to exclude unnecessary build context
5. Consider multi-stage build to exclude build tools from final image
6. Remove package manager caches in the same RUN that installs them

In our project when image size became a concern, I first checked
which layer was largest using: docker history banking-app:42
This shows size per layer — helps identify which instruction
is adding the most weight.

---

## PART 6: QUICK REFERENCE CARD (review before interview)

```
Dockerfile format:     INSTRUCTION argument
Location:              Root of GitHub repository
Built by:              docker build -t image-name:tag .
The . means:           Use Dockerfile in current folder

RUN   = BUILD TIME   = baked into image = for setup/install
CMD   = STARTUP TIME = default command  = can be overridden
ENTRYPOINT = STARTUP = mandatory command = cannot be overridden

Layer caching rule:    Stable layers FIRST, changing layers LAST
COPY vs ADD:           Always use COPY unless you need auto-extraction
EXPOSE:                Documentation only — -p actually opens the port
Image vs Container:    Image = recipe, Container = cake made from recipe

Reduce image size:
  1. Minimal base image (Alpine variants)
  2. Combine RUN commands
  3. .dockerignore file
  4. Multi-stage builds
  5. Clean up in same RUN as install

Troubleshooting commands:
  docker build -t name:tag .         Build image
  docker images                      List images (check size)
  docker history image-name:tag      Show layers and sizes
  docker ps                          List running containers
  docker ps -a                       List all containers (including stopped)
  docker logs <container>            See container output/errors
  docker exec -it <container> bash   Get shell inside container
  docker system df                   Show Docker disk usage
  docker system prune -f             Clean unused images/containers
```
