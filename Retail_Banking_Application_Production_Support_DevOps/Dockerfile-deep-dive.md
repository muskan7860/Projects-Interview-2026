# Dockerfile — Every Line Explained (Zero to Understanding)

---

## BEFORE WE READ — what is a Dockerfile and where does it live?

Remember Stage 4 in the Jenkinsfile?
`docker build -t banking-app:42 .`
The `.` at the end means: "look in the CURRENT FOLDER for a file
called 'Dockerfile' and use it as the instructions."

A Dockerfile is a RECIPE. It tells Docker:
"To build my application's image, follow these steps in order."

**Where does it live?**: In the root of the GitHub repository,
right next to the application code and the Jenkinsfile.
All three files travel together in the same repo.

**Who writes it?**: In our team, the DevOps engineer writes and
maintains the Dockerfile. Developers write the application code.
We write the "how to package and run it" instructions.

**Analogy**: Your application code is the CAKE INGREDIENTS.
The Dockerfile is the RECIPE for baking the cake.
The Docker image is the BAKED CAKE (ready to serve).
The Docker container is the SLICE OF CAKE being served to a customer.

---

## HOW DOCKER BUILDS AN IMAGE — concept first

When you run `docker build`, Docker reads the Dockerfile top to bottom.
Each instruction (FROM, RUN, COPY, EXPOSE, CMD) creates a new LAYER.

Think of layers like building a house:
- Layer 1: Foundation (base OS)
- Layer 2: Walls (Java runtime)
- Layer 3: Roof (Tomcat web server)
- Layer 4: Interior (our banking app WAR file)
- Layer 5: Finishing (settings, exposed ports)

Each layer is CACHED by Docker. If a layer hasn't changed since
last build, Docker reuses the cached version instead of rebuilding.
This makes subsequent builds MUCH faster — only changed layers rebuild.

The final image = all layers stacked together = one complete package.

---

## THE COMPLETE DOCKERFILE — every line explained

```dockerfile
FROM tomcat:9.0-jre17
```

**What FROM means**: "Start with THIS existing image as the base."
This is ALWAYS the first line of any Dockerfile.
You're not building from absolute zero — you're standing on
the shoulders of someone else's work.

**What is 'tomcat:9.0-jre17'?**
This is an official, pre-built image maintained by the Apache Tomcat team.
It already contains:
- A minimal Linux operating system (Alpine or Debian based)
- Java JRE 17 (the runtime that executes Java code)
- Apache Tomcat 9.0 (the web server that hosts WAR files)

**Why not install Java and Tomcat ourselves from scratch?**
We could. But:
- It would take 20+ lines of complex commands
- We'd have to update it ourselves every time Java releases a patch
- More lines = more chance of mistakes
- The official image is maintained by experts, tested, and secure

By using `FROM tomcat:9.0-jre17`, we get a fully working Java+Tomcat
environment in ONE line. We just add our application on top.

**The ':9.0-jre17' part** = the TAG of the Tomcat image.
9.0 = Tomcat version 9.0 (the version number)
jre17 = includes Java 17 JRE (Java Runtime Environment)
Always specify an exact version tag — never use 'latest' for base
images in production. Why? 'Latest' changes when a new version
is released — your image might suddenly use Tomcat 10 without
you explicitly choosing to upgrade. Pin to a specific version
for stability and reproducibility.

**Behind the scenes when Docker processes this line**:
1. Docker checks if 'tomcat:9.0-jre17' exists locally (cached)
2. If not, downloads it from Docker Hub automatically
3. Uses it as the starting point — Layer 1 of our image

---

```dockerfile
RUN rm -rf /usr/local/tomcat/webapps/ROOT
```

**What RUN means**: "Execute this shell command INSIDE the container
while building the image."
This command runs during the BUILD process, not when the container starts.

**What this specific command does**:
`rm` = remove
`-rf` = recursive (including folders inside) and force (don't ask "are you sure?")
`/usr/local/tomcat/webapps/ROOT` = a specific folder path inside Tomcat

**Why do we delete this?**
When you install Tomcat, it comes with a DEFAULT welcome page
stored in a folder called ROOT.
This welcome page appears when you visit port 8080.
It says "Welcome to Tomcat!" — Apache's default page, not your app.

If we don't delete it, our banking app and Tomcat's default page
would conflict — both trying to be the "main" application.
By deleting ROOT, we clear space for our application to take over
as the root application (accessible at the main URL, not a sub-path).

**Behind the scenes**:
This command runs inside a temporary container during the build.
The result (ROOT folder deleted) becomes a new layer in our image.
Every container started from our image will have ROOT already deleted.

---

```dockerfile
COPY target/banking-app.war /usr/local/tomcat/webapps/ROOT.war
```

**What COPY means**: "Copy a file FROM the build context
(the folder on the Jenkins server where the build is running)
INTO the image."

Format: `COPY <source on Jenkins server> <destination inside image>`

**Source**: `target/banking-app.war`
This is the WAR file that Maven created in Stage 2 (the Build stage).
It exists on the Jenkins server at this path relative to the repo root.
`target/` is the folder Maven creates during build.
`banking-app.war` is the packaged application.

**Destination**: `/usr/local/tomcat/webapps/ROOT.war`
This is the special folder where Tomcat looks for web applications.
By naming it ROOT.war (instead of banking-app.war), we tell Tomcat:
"this is the ROOT application" — accessible at `http://server:8080/`
(not at `http://server:8080/banking-app/`).

**What Tomcat does with ROOT.war automatically**:
When Tomcat starts, it watches the `webapps/` folder.
When it finds a .war file, it AUTOMATICALLY:
1. Extracts/unpacks it
2. Deploys it as a web application
3. Makes it accessible on port 8080

You don't need to tell Tomcat "please deploy this" — it does it
automatically just by placing the file in the right folder.

**Behind the scenes**:
Docker copies the WAR file into the image at build time.
Every container started from this image already has the WAR file
inside Tomcat's webapps folder — ready for Tomcat to deploy on startup.

---

```dockerfile
EXPOSE 8080
```

**What EXPOSE means**: "Document that this container listens
on port 8080."

Important: EXPOSE does NOT actually open the port or make it
accessible from outside. It's documentation — telling Docker
and people reading the Dockerfile: "this container uses port 8080."

**Then why write it?**
1. Documentation — anyone reading this Dockerfile immediately knows
   which port the app uses. Critical for operations teams.
2. Required for some Docker tools and orchestration systems
   (like Kubernetes) that read EXPOSE to configure networking.
3. The ACTUAL port mapping (`-p 8080:8080`) happens in deploy.sh
   when we run the container — that's what really opens the port.

**Analogy**: EXPOSE is like writing "front door" on a blueprint.
The `-p` flag in `docker run` is like actually building the door.

---

```dockerfile
CMD ["catalina.sh", "run"]
```

**What CMD means**: "When a container is STARTED from this image,
run THIS command automatically."

This is the STARTUP COMMAND — what runs when someone does `docker run`.

**What is 'catalina.sh run'?**
`catalina.sh` is Tomcat's startup script.
`run` = start Tomcat in the FOREGROUND (not background).

Why foreground? Docker containers stay alive only as long as their
main process is running. If we started Tomcat in the background
and the CMD finished, the container would immediately stop.
By running Tomcat in the foreground, the container stays alive
as long as Tomcat is running.

**Behind the scenes when someone runs `docker run ... banking-app:42`**:
1. Docker creates a new container from the image
2. Starts the container
3. Automatically runs `catalina.sh run` inside
4. Tomcat starts up
5. Tomcat finds ROOT.war in webapps/
6. Tomcat deploys it
7. Application becomes available on port 8080
8. Container stays alive because Tomcat (foreground process) is running

**CMD vs ENTRYPOINT (common interview question)**:
CMD = the default command, CAN be overridden when running the container.
Example: `docker run banking-app:42 bash` would start bash instead of Tomcat.
ENTRYPOINT = the command that ALWAYS runs, cannot be easily overridden.
For application containers, CMD is most common.

---

## THE COMPLETE DOCKERFILE — all 5 lines together

```dockerfile
# Start from an official image that has Java 17 + Tomcat 9.0 pre-installed
FROM tomcat:9.0-jre17

# Remove Tomcat's default welcome page to avoid conflicts with our app
RUN rm -rf /usr/local/tomcat/webapps/ROOT

# Copy our built banking application into Tomcat's deployment folder
COPY target/banking-app.war /usr/local/tomcat/webapps/ROOT.war

# Document that this container uses port 8080
EXPOSE 8080

# When the container starts, start Tomcat (which auto-deploys our app)
CMD ["catalina.sh", "run"]
```

5 lines. That's the complete recipe for running our banking application.

---

## HOW THESE 5 LINES CONNECT TO EVERYTHING ELSE

```
Stage 2 (Build) creates:   target/banking-app.war
                                        ↓
Stage 4 (Docker Build) reads: Dockerfile
  Line 1: FROM → downloads base image (Java + Tomcat pre-installed)
  Line 2: RUN  → deletes default Tomcat page
  Line 3: COPY → puts banking-app.war inside the image
  Line 4: EXPOSE → documents port 8080
  Line 5: CMD  → sets startup command
  Result: banking-app:42 image created
                                        ↓
Stage 5 (Docker Push): banking-app:42 uploaded to Docker Hub
                                        ↓
deploy.sh (docker run):
  -p 8080:8080 → ACTUALLY opens port 8080 (EXPOSE just documented it)
  Container starts → CMD runs → Tomcat starts → finds ROOT.war → deploys app
  App available at http://server-ip:8080
                                        ↓
ALB health check: http://server:8080/health → 200 OK
ALB adds server to target group → customer traffic reaches our app
```

---

## LAYER CACHING — why this matters for speed

```
Layer 1: FROM tomcat:9.0-jre17      ← rarely changes (weeks/months)
Layer 2: RUN rm -rf ...webapps/ROOT  ← never changes
Layer 3: COPY target/banking-app.war ← changes EVERY build (new code!)
Layer 4: EXPOSE 8080                 ← never changes
Layer 5: CMD [...]                   ← never changes
```

Because of Docker's layer caching:
- Layers 1, 2 = cached after first build → reused in all future builds
- Layer 3 = new WAR file = new layer = must rebuild
- Layers 4, 5 = cached → reused

Result: After the first build, subsequent builds are FAST because
Docker only rebuilds Layer 3 (the COPY instruction with the new WAR file)
and everything after it. Layers 1 and 2 are pulled from cache instantly.

**Interview line**: "We structured the Dockerfile to put frequently
changing instructions (the COPY of the WAR file) after the
rarely-changing instructions (FROM, RUN). This maximizes Docker's
layer cache efficiency — only the layers that actually changed
get rebuilt, making builds significantly faster."

---

## KEY INTERVIEW POINTS FROM THIS DOCKERFILE

- FROM = base image → we build ON TOP of existing Java+Tomcat image,
  not from scratch → faster to write, maintained by experts
- Always pin base image to specific version (9.0-jre17), never 'latest'
- RUN executes during BUILD time (not container runtime)
- COPY puts files FROM the build context INTO the image
- ROOT.war naming → Tomcat auto-deploys at root URL (not sub-path)
- EXPOSE = documentation only → actual port opening is in deploy.sh (-p)
- CMD = startup command → runs when container starts
- Layer order matters → stable layers first → frequently changing last
  → maximizes cache efficiency
- 5 lines is ALL you need to package a complete Java web application
- Same image runs in QA and Production → config differs via -e flags
  in deploy.sh, not in the Dockerfile → one image, multiple environments
