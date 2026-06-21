# Mock Interview — Dockerfile & Docker Q&A (Practiced Session)

---

## HOW TO USE THIS FILE
- Read the question first
- CLOSE the answer section
- Try answering out loud from memory
- Then open and compare
- Practice until you can answer WITHOUT reading

---

## QUESTION 1: What is a Dockerfile and why do we use it?

### Your original answer was:
Good — you got the core idea (compatibility problem, wrapping code).
Needed fix: say "Docker image" not "one file."

### Polished Answer (practice saying this):
"A Dockerfile is a text file containing instructions for building
a Docker image. We use it to solve the 'works on my machine' problem.

When a developer writes code on their laptop with a specific Java
version and we deploy it to 3 different servers, each server might
have different versions installed and things break.

With a Dockerfile, we package the application code AND all its
dependencies — the exact Java version, Tomcat, everything — into
one Docker image. That image runs identically on any server, any
machine, without compatibility issues.

In our banking project, this was especially important because we
deployed to 3 EC2 instances — the same Docker image ran on all 3,
guaranteeing identical behaviour across all servers."

---

## QUESTION 2: What is the difference between RUN, CMD, and ENTRYPOINT?

### Your original answer was:
Very good — you got all three correct.
Missing: exact path for RUN, CMD example, why foreground for CMD.

### Polished Answer (practice saying this):
"RUN executes during image BUILD time — when you run docker build.
The result is baked permanently into the image as a layer.
In our Dockerfile, we used RUN to delete Tomcat's default welcome
page at /usr/local/tomcat/webapps/ROOT — because our banking app
and that default page would conflict on port 8080.
This deletion happens once at build time and is permanent in the image.

CMD executes at CONTAINER STARTUP — every time a container is
created from the image. It can be overridden — if someone passes
a different command to docker run, CMD is replaced.
In our Dockerfile, CMD is 'catalina.sh run' — this starts Tomcat
in the FOREGROUND every time the container starts.
Foreground is important — if Tomcat ran in the background,
the container would die immediately because Docker stops
when the main process finishes.

ENTRYPOINT also runs at container startup but CANNOT be overridden —
it always runs no matter what command is passed to docker run.
We didn't use ENTRYPOINT in our Dockerfile because CMD was sufficient."

### Quick memory trick:
```
RUN         = BUILD time   = construction worker (builds house ONCE)
CMD         = STARTUP time = morning routine (runs daily, guest can override)
ENTRYPOINT  = STARTUP time = hotel rule (always runs, nobody can skip)
```

---

## QUESTION 3: What is Docker layer caching and how did it help your pipeline?

### Your original answer was:
Good — core concept correct.
Needed fix: be specific about WHICH layers cache and which don't.

### Polished Answer (practice saying this):
"When Docker builds an image, each instruction in the Dockerfile
creates a layer. Docker caches every layer after it's built.
On the next build, Docker checks each instruction — if the
instruction hasn't changed and the files it depends on haven't
changed, Docker reuses the cached layer instead of rebuilding it.

In our Dockerfile:
- FROM layer — never changes — always pulled from cache instantly
- RUN layer — never changes — always pulled from cache instantly
- COPY layer — changes EVERY build (new WAR file with new code)
  — always rebuilds
- Everything after COPY — also rebuilds

The real benefit in our pipeline: first build took around 3 minutes.
Every build after took about 30 seconds — because only the COPY
layer and below rebuild. FROM and RUN are instant from cache.

We also structured our Dockerfile to put stable, rarely-changing
instructions FIRST and frequently-changing instructions LAST —
this maximizes cache efficiency."

### Key rule to remember:
Stable layers FIRST → frequently changing layers LAST
= maximum cache reuse = fastest builds

---

## QUESTION 4: If your Docker image size became too large, what would you do?

### Your original answer was:
Good — you remembered 3 points AND caught the non-root user point
that wasn't in the notes. Strong answer.

### Polished Answer (practice saying this):
"If our Docker image size became too large, I'd approach it in order:

First, check which layers are contributing most using
'docker history banking-app:42' — shows size per layer
so I know where to focus.

Second, use a minimal base image. Switching from tomcat:9.0-jre17
to tomcat:9.0-jre17-alpine significantly reduces size —
Alpine Linux is about 5MB compared to Debian's 120MB base.
Same Java and Tomcat, much smaller OS.

Third, add a .dockerignore file to exclude unnecessary files
from the build context — test files, documentation, .git folder,
local config files. These don't belong in the image and also
slow down the build.

Fourth, combine RUN commands. Each RUN creates a layer.
Instead of three separate RUN instructions for update, install,
and clean — combine them into one RUN with && between commands.
One layer instead of three, and cleanup actually removes files
from that same layer.

Fifth, run as non-root user — not about size directly but a
security best practice, especially important in banking.
We create a dedicated application user and use the USER instruction
before CMD, so the container never runs as root.

If we needed to go further, multi-stage builds — use one stage
with full Maven and JDK to compile, then copy only the WAR file
into a minimal runtime stage. Build tools never make it
into the final image, which can reduce size dramatically."

### Non-root user — code to remember:
```dockerfile
RUN groupadd -r appuser && useradd -r -g appuser appuser
RUN chown -R appuser:appuser /usr/local/tomcat
USER appuser
CMD ["catalina.sh", "run"]
```

### Commands for checking image size:
```bash
docker images                    # shows total image size
docker history banking-app:42    # shows size per layer
docker system df                 # shows total Docker disk usage
```

---

## QUESTION 5: Container starts but app not reachable on port 8080 — troubleshoot it

### Your original answer:
You said you didn't know this one — we learned it together.
Now practice the answer below until it's automatic.

### The troubleshooting chain (memorize this order):
```
App not reachable on port 8080
            ↓
STEP 1: docker ps
        → Is container running?
        → Does PORTS column show 0.0.0.0:8080->8080/tcp?
            ↓
Container NOT running:
        → docker ps -a (check exit code)
        → Exit 1   = app error
        → Exit 137 = killed by OS (out of memory)
        → Exit 139 = serious crash
        → docker logs banking-app (see exact error)
            ↓
Container running but NO port mapping:
        → Container started without -p 8080:8080 flag
        → Stop and restart with correct -p flag
            ↓
Container running AND port mapped correctly:
        → docker logs banking-app (look for errors inside)
        → DB connection failure? Wrong config?
        → Java OutOfMemoryError?
        → Still initializing?
            ↓
No obvious error in logs:
        → docker exec -it banking-app bash
        → curl http://localhost:8080/health FROM INSIDE
            ↓
Works from INSIDE, not from OUTSIDE:
        → AWS Security Group issue
        → ALB can't reach port 8080 on app server
        → Check SG rules
            ↓
Fails from INSIDE too:
        → App itself is broken
        → Check DB connectivity, memory limits, config
```

### Polished Interview Answer (practice saying this):
"First I'd run docker ps to confirm the container is running
and check if port mapping shows correctly — 0.0.0.0:8080->8080/tcp.

If the container isn't running, I'd check docker ps -a for the
exit code and docker logs to see why it crashed — exit code 1
means app error, 137 means killed by the OS due to out of memory.

If the container is running but port mapping is missing, the
container was started without the -p 8080:8080 flag — I'd
stop it and restart correctly.

If container is running and port is mapped correctly, I'd check
docker logs banking-app for errors inside — database connection
failures, Java errors, or Tomcat startup issues.

Then I'd exec into the container with docker exec -it banking-app bash
and curl localhost:8080 directly from inside the container.

If it works from inside but not from outside, it's a Security Group
issue at the AWS level — the ALB can't reach port 8080 on the app
server and I'd check the security group rules.

If it fails from inside too, the application itself has a problem —
wrong database config, out of memory, or the app is still initializing."

---

## BONUS QUESTIONS — practice these too

### Q6: What is the difference between a Docker image and a container?
"An image is a read-only template — the blueprint or recipe.
A container is a running instance created FROM that image.
The relationship: image is like a recipe, container is the actual
cake made from that recipe. You can run many containers from
the same image simultaneously — like baking many cakes from
one recipe. In our project, the same banking-app:42 image
ran as 3 separate containers on 3 separate EC2 instances."

---

### Q7: What is .dockerignore and why did you use it?
".dockerignore works like .gitignore — it tells Docker which files
to EXCLUDE from the build context when running docker build.
Without it, Docker sends the entire project folder to the build
process, including test files, documentation, .git history,
and local config files that are unnecessary in the final image.

In our project we excluded test files, documentation, .git folder,
and local environment configs. This reduced our build context size,
made docker build faster, and kept unnecessary files out of the image."

---

### Q8: What is Docker Compose and did you use it?
"Docker Compose is a tool for defining and running multi-container
applications using a single YAML file called docker-compose.yml.
Instead of running multiple docker run commands manually for each
container, you define all containers, their configs, and how they
connect in one file. Then one command — docker-compose up — starts
everything together in the right order.

In our project, we used Docker Compose for LOCAL DEVELOPMENT AND
TESTING only — developers could spin up the full banking app plus
SQL Server database on their laptops with one command, matching
the production setup closely.

In production, we deployed individual containers via our Jenkins
pipeline and deploy.sh script, not Docker Compose."

---

### Q9: What is a multi-stage build?
"Multi-stage builds use multiple FROM instructions in one Dockerfile.
Each FROM starts a new build stage. You can copy artifacts from
one stage to another, leaving behind everything only needed for building.

For a Java application:
Stage 1 uses full Maven plus JDK image to compile code and produce WAR.
Stage 2 uses minimal JRE plus Tomcat image and copies only the WAR file.

Final image contains only JRE, Tomcat, and our WAR file.
No Maven, no JDK, no source code, no build tools in production.
This can reduce image size from 600MB to under 100MB.

In our project we didn't implement multi-stage builds during my time,
but I'm aware of the pattern and it was discussed as a future
optimization for reducing our image size."

---

### Q10: What happens if two RUN commands can be combined — why does it matter?
"Each RUN instruction creates a separate layer in the image.
If you install software in one RUN and clean up cache in another RUN,
the cleanup creates a new layer but the cache files are still captured
in the previous install layer — the image size doesn't actually reduce.

By combining them in ONE RUN with && between commands:
RUN apt-get update && apt-get install -y curl && apt-get clean

Everything happens in one layer. The cleanup removes files within
the same layer, actually reducing the final image size.
Also one layer instead of three means a smaller, cleaner image
with fewer layers to pull when deploying."

---

## QUICK REFERENCE — commands to know cold

```bash
# Build image
docker build -t banking-app:42 .

# Run container
docker run -d --name banking-app -p 8080:8080 banking-app:42

# List running containers
docker ps

# List ALL containers including stopped
docker ps -a

# Check container logs
docker logs banking-app
docker logs banking-app --tail 50    # last 50 lines only

# Go inside a running container
docker exec -it banking-app bash

# Check image sizes
docker images
docker history banking-app:42        # size per layer

# Check Docker disk usage
docker system df

# Clean up unused images and containers
docker system prune -f
docker image prune -f

# Stop and remove container
docker stop banking-app
docker rm banking-app

# Push image to Docker Hub
docker login
docker tag banking-app:42 yourteam/banking-app:42
docker push yourteam/banking-app:42

# Pull image from Docker Hub
docker pull yourteam/banking-app:42
```
