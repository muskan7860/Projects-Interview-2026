# Dockerfile — Complete Guide (Zero Knowledge to Interview Ready)

---

## THE PROBLEM DOCKER SOLVES (why it exists)

Imagine you built a banking application on YOUR laptop. It works perfectly.
Your manager says: "Deploy this on our 3 AWS servers."

You go to Server 1:
- Install Java... Server has Java 11, you used Java 17. App breaks.
- Install Tomcat... wrong version. App breaks.
- Copy files... folder structure different. App breaks.

2 days wasted on Server 1. Same problems on Server 2. Server 3.
Classic problem: "It works on my machine!"

Docker solution: Pack the application AND everything it needs
(correct Java version, correct Tomcat, correct folder structure)
into ONE sealed box. That box runs IDENTICALLY anywhere.

Sealed box = Docker container
Recipe for building that box = Dockerfile

---

## WHAT IS A WAR FILE? (simplest explanation)

When you send many files to someone, you ZIP them.
Instead of 500 separate files, you send 1 ZIP file.

A WAR file is exactly that — but for Java web applications.

Developers write the banking app in hundreds of files:
- .java files (logic: "check balance", "transfer money")
- HTML/CSS files (web pages customers see)
- Config files (settings, database address)

Maven (build tool, Stage 2) packs ALL of these into ONE file:
banking-app.war

WAR = Web Application Archive = ZIP file Tomcat understands

Think of it like:
- Clothes scattered around room = hundreds of Java files
- Packed suitcase = the WAR file
- Hotel (Tomcat) = unpacks suitcase and arranges everything properly

---

## WHAT IS TOMCAT? (simplest explanation)

Tomcat is a web server — listens on port 8080, receives requests,
runs Java code, sends back responses.

Think of it as a restaurant kitchen:
- Customer orders food (HTTP request from browser)
- Kitchen receives order (Tomcat receives request)
- Kitchen cooks food (Tomcat runs Java code)
- Food delivered to customer (Tomcat sends response back)

Tomcat's special behaviour:
If you put a WAR file into its webapps/ folder,
it automatically unpacks it and serves that application.
No manual steps — just put the WAR file in the right place.

---

## WHAT ARE LAYERS? (house building analogy)

Building a house happens in stages:
1. Pour the foundation
2. Build the walls
3. Put on the roof
4. Paint the walls
5. Put furniture inside

Each stage builds on top of the previous one.
You can't paint before walls exist.
You can't add furniture before the roof.

Docker images work exactly the same way.
Each Dockerfile instruction adds one layer on top of previous:

```
Layer 5: CMD    — startup command
Layer 4: EXPOSE — document port 8080
Layer 3: COPY   — put WAR file inside     ← changes EVERY build
Layer 2: RUN    — delete default page
Layer 1: FROM   — Java + Tomcat foundation
```

THE MAGIC — CACHING:
Once Layer 1 is built, Docker saves it (caches it).
Next build: Layer 1 unchanged → Docker reuses saved version. Skip.
Layer 3 (WAR file) changes every build → Docker rebuilds from here.
Layers 1, 2 = reused from cache instantly.

Result:
First build: 3 minutes
Every build after: 30 seconds (only changed layers rebuild)

---

## THE DOCKERFILE — 5 LINES, EVERY LINE EXPLAINED

```dockerfile
FROM tomcat:9.0-jre17
RUN rm -rf /usr/local/tomcat/webapps/ROOT
COPY target/banking-app.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

---

### LINE 1: FROM tomcat:9.0-jre17

ANALOGY:
You want to bake a cake. Two options:
- Option A: Grow wheat, grind flour, make everything from scratch (weeks)
- Option B: Buy ready-made cake base, just add your toppings (minutes)

FROM = Option B.

Someone at Apache already built a Docker image containing:
- Small Linux operating system
- Java 17 installed correctly
- Tomcat 9.0 installed correctly
- Everything configured properly

We say: "Start with THEIR work as our foundation."

tomcat     = name of pre-built image on Docker Hub
9.0        = Tomcat version 9.0 (exact version, not "latest")
jre17      = Java Runtime Environment 17 included

WHY PIN EXACT VERSION (9.0-jre17) NOT "latest"?
"latest" changes when a new version releases.
Today: latest = Tomcat 9. Tomorrow: latest = Tomcat 10.
Your app might not be compatible with Tomcat 10.
Build worked yesterday, breaks today — confusing and dangerous.
Exact version = same base image every single time. Reproducible.

BEHIND THE SCENES:
Docker downloads this pre-built image from Docker Hub (if not cached).
Becomes Layer 1 of our image.
We got Java + Tomcat for free — without writing a single install command.

INTERVIEW ANSWER:
"We use an official Tomcat base image with Java 17 pre-installed.
We don't manually install and configure Java and Tomcat — we inherit
a tested, maintained, production-ready base. We pin the exact version
rather than 'latest' to ensure reproducible builds — same base image
every time, no surprise upgrades that could break our application."

---

### LINE 2: RUN rm -rf /usr/local/tomcat/webapps/ROOT

THE PROBLEM:
Fresh Tomcat installation comes with a default welcome page
stored in a folder called ROOT.
When you visit http://server:8080, you see Apache's "Welcome to Tomcat!" page.
Not your banking app — Apache's default page.

If you put your banking app in the same place as the default page,
they conflict. So delete the default page first. Clean slate.

WHAT EACH PART MEANS:
RUN    = "during image BUILD, run this Linux command"
rm     = remove (delete)
-r     = recursive — delete the folder AND everything inside it
-f     = force — don't ask "are you sure?" just delete
/usr/local/tomcat/webapps/ROOT = exact path of default page folder

IMPORTANT: RUN executes during BUILD TIME (when you run docker build).
NOT when the container starts. The deletion is baked into the image.
Every container from this image has ROOT already deleted permanently.

BEHIND THE SCENES:
Docker runs this command inside a temporary container during build.
ROOT folder deleted. Saved as Layer 2.
We never have to delete it again — it's gone from all future containers.

INTERVIEW ANSWER:
"We remove Tomcat's default ROOT webapp to avoid conflicts.
When we deploy our banking application as ROOT.war, it becomes
the root application, accessible at the base URL without any sub-path.
Without this step, there would be a conflict between our app
and Tomcat's default welcome page."

---

### LINE 3: COPY target/banking-app.war /usr/local/tomcat/webapps/ROOT.war

THIS IS THE MOST IMPORTANT LINE.
This is where YOUR application enters the image.

COPY = "take a file from OUTSIDE (Jenkins server) and put it INSIDE (image)"

LEFT SIDE: target/banking-app.war
- Location: on the Jenkins server
- This is the WAR file Maven created in Stage 2 (Build stage)
- target/ = folder Maven creates for build outputs
- banking-app.war = our complete packaged banking application

RIGHT SIDE: /usr/local/tomcat/webapps/ROOT.war
- Location: inside the image being built
- /usr/local/tomcat/webapps/ = Tomcat's watched folder for applications
- ROOT.war = we specifically name it ROOT.war (important — see below)

WHY NAME IT ROOT.war NOT banking-app.war?
Tomcat uses the filename to determine the URL path:
- banking-app.war → app at http://server:8080/banking-app/
- ROOT.war        → app at http://server:8080/  (root URL, no sub-path)

Customers type bankportal.com — not bankportal.com/banking-app.
ROOT.war = app at root URL = correct behaviour.

WHAT TOMCAT DOES AUTOMATICALLY WHEN IT FINDS ROOT.war:
1. Unzips/extracts it into a folder called ROOT
2. Reads application configuration
3. Starts serving the application on port 8080
All automatic — you don't tell Tomcat anything extra.

LAYER CACHING IMPACT:
This layer changes EVERY build (new code = new WAR file = new layer).
Layers 1 and 2 are reused from cache.
Only this layer and below get rebuilt. That's why builds are fast.

INTERVIEW ANSWER:
"We COPY the WAR file built by Maven into Tomcat's webapps directory,
naming it ROOT.war so Tomcat serves it at the root URL.
This happens at image build time — every container started from
this image already has the application pre-loaded.
Tomcat automatically detects and deploys ROOT.war on startup
without any additional configuration."

---

### LINE 4: EXPOSE 8080

ANALOGY:
An architect draws a building blueprint.
They draw where the doors go.
Drawing the door on paper doesn't BUILD the door — it shows "door goes here."

EXPOSE 8080 = drawing the door on the blueprint.
It says "this container has a door on port 8080."
It does NOT actually open the port.

THE ACTUAL DOOR OPENING happens in deploy.sh:
docker run -p 8080:8080 ...
This -p flag is what really connects outside traffic to inside the container.

WHY WRITE EXPOSE IF IT DOESN'T OPEN THE PORT?
1. Documentation — anyone reading Dockerfile knows: "this app uses 8080"
2. Kubernetes and other tools read EXPOSE to configure networking
3. Standard professional practice — every real Dockerfile has it
4. Makes the intent clear to your team

INTERVIEW ANSWER:
"EXPOSE 8080 documents that our application listens on port 8080
inside the container. The actual port binding to the host happens
with the -p flag in our deploy.sh when we run the container —
EXPOSE itself doesn't open anything. It's documentation for humans
and a signal for container orchestration tools."

---

### LINE 5: CMD ["catalina.sh", "run"]

ANALOGY:
You assembled a car completely.
What makes it actually START?
You turn the ignition key.

CMD is the ignition key of the container.
It defines: "when someone starts a container from this image,
automatically run THIS command — without anyone typing anything."

catalina.sh = Tomcat's startup script
              Lives at /usr/local/tomcat/bin/catalina.sh
              Comes pre-installed in our base image (FROM tomcat:9.0-jre17)
run         = start Tomcat in the FOREGROUND

WHY FOREGROUND AND NOT BACKGROUND?
Docker containers live as long as their MAIN PROCESS lives.

If Tomcat runs in background:
CMD finishes immediately (nothing left running in foreground).
Docker thinks: "main process ended" → kills the container.
Container dies instantly after starting. App never serves traffic.

If Tomcat runs in foreground:
Tomcat process stays active, occupying the foreground.
Container stays alive as long as Tomcat runs.
App serves traffic continuously. Correct behaviour.

WHAT HAPPENS WHEN deploy.sh RUNS docker run:
Step 1: Docker creates container from banking-app:42 image
Step 2: Container starts
Step 3: CMD fires automatically: catalina.sh run
Step 4: Tomcat starts up (takes 10-15 seconds)
Step 5: Tomcat scans webapps/ folder, finds ROOT.war
Step 6: Tomcat automatically extracts and deploys ROOT.war
Step 7: Banking app is live on port 8080
Step 8: Container stays alive (Tomcat running in foreground)
Step 9: deploy.sh health check: curl http://localhost:8080/health → 200 OK
Step 10: ALB detects healthy server → sends customer traffic here

INTERVIEW ANSWER:
"CMD defines the startup command that runs automatically when
a container is started. We use catalina.sh run to start Tomcat
in the foreground — foreground keeps the container alive as long
as Tomcat runs. When Tomcat starts, it automatically detects
our ROOT.war file in the webapps directory and deploys it,
making our banking application available immediately on port 8080."

---

## THE COMPLETE PICTURE — all 5 lines connected

```
BEFORE DOCKERFILE RUNS:
Jenkins server has: target/banking-app.war (Maven built this in Stage 2)

docker build -t banking-app:42 . (Stage 4 in Jenkinsfile triggers this)
                    ↓
Line 1: FROM tomcat:9.0-jre17
        Download base: Linux + Java 17 + Tomcat 9.0
        Layer 1 created (cached after first build)
                    ↓
Line 2: RUN rm -rf .../webapps/ROOT
        Delete Tomcat's default welcome page
        Layer 2 created (cached — never changes)
                    ↓
Line 3: COPY target/banking-app.war .../webapps/ROOT.war
        Take WAR file from Jenkins server
        Put it inside the image in Tomcat's folder
        Layer 3 created (NEW every build — new code = new WAR)
                    ↓
Line 4: EXPOSE 8080
        Document that port 8080 is used
        Layer 4 created (cached — never changes)
                    ↓
Line 5: CMD ["catalina.sh", "run"]
        Set: "when container starts, run Tomcat"
        Layer 5 created (cached — never changes)
                    ↓
RESULT: banking-app:42 image exists on Jenkins server
= sealed box containing:
  Linux OS + Java 17 + Tomcat 9.0 + banking-app.war + startup command
= runs IDENTICALLY on Server 1, Server 2, Server 3, or anywhere

NEXT: Stage 5 pushes this to Docker Hub
NEXT: deploy.sh pulls it on each server and runs it
NEXT: CMD fires → Tomcat starts → finds ROOT.war → deploys it
NEXT: App live on port 8080
NEXT: ALB health check passes → customer traffic flows
```

---

## IF INTERVIEWER SAYS "WRITE THE DOCKERFILE AND EXPLAIN IT"

Write this (from memory — practice until automatic):

```dockerfile
FROM tomcat:9.0-jre17
RUN rm -rf /usr/local/tomcat/webapps/ROOT
COPY target/banking-app.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

Say while writing:
"I'm starting with an official Tomcat base image that has Java 17
pre-installed — no need to install Java and Tomcat from scratch,
and I pin the exact version for reproducibility.

Then I delete Tomcat's default ROOT webapp to avoid conflicts
with our application.

I copy the WAR file that Maven built into Tomcat's webapps directory,
naming it ROOT.war so it's served at the root URL without any sub-path.

EXPOSE documents that port 8080 is used — the actual port binding
happens in our deployment script with -p 8080:8080.

Finally, CMD starts Tomcat in the foreground — foreground is important
so the container stays alive as long as Tomcat runs.

When a container starts from this image, Tomcat automatically finds
ROOT.war and deploys our banking application — no manual steps."

---

## QUICK INTERVIEW Q&A — Dockerfile specific

Q: What is the difference between RUN and CMD?
A: RUN executes during image BUILD time — the result is baked
   into the image as a layer. CMD executes at CONTAINER STARTUP
   time — it's the command that runs when someone does docker run.
   RUN = build time. CMD = runtime.

Q: What is the difference between CMD and ENTRYPOINT?
A: CMD defines a default command that CAN be overridden when
   running the container (docker run myimage bash would run bash
   instead of CMD). ENTRYPOINT defines a command that ALWAYS runs
   and is harder to override. For application containers like ours,
   CMD is the standard choice.

Q: Why use a base image instead of building from scratch?
A: Installing Java and Tomcat from scratch takes 20+ lines of
   complex commands, and we'd be responsible for keeping it
   updated and secure. Official base images are maintained by
   the tool's own team, regularly patched, and widely tested.
   We focus on our application, not on maintaining Java installations.

Q: How does Docker layer caching help in CI/CD?
A: Layers that don't change between builds are reused from cache.
   In our Dockerfile, the FROM and RUN layers never change —
   they're cached after the first build. Only the COPY layer
   (new WAR file) and layers after it rebuild each time.
   This reduces our Docker build time from minutes to seconds
   on every subsequent pipeline run.

Q: What happens if you don't use EXPOSE?
A: The application still works if you use -p in docker run.
   EXPOSE is documentation — skipping it doesn't break anything
   but makes the Dockerfile harder to understand and may cause
   issues with Kubernetes or other tools that rely on it.

Q: Why name the WAR file ROOT.war specifically?
A: Tomcat uses the WAR filename as the URL context path.
   ROOT.war = served at http://server:8080/ (root URL).
   Any other name (e.g. banking-app.war) would require customers
   to access http://server:8080/banking-app/ — not acceptable
   for a production banking portal.

---

## KEY THINGS TO REMEMBER (review before interview)

1. Dockerfile = recipe for building Docker image
2. FROM = base image = foundation (Java + Tomcat pre-installed)
3. Always pin version — never use :latest for base images
4. RUN = runs during BUILD. CMD = runs when CONTAINER STARTS
5. WAR file = Maven's output = entire app packed in one file
6. ROOT.war = served at root URL (/) not sub-path (/banking-app)
7. EXPOSE = documentation only, NOT actual port opening
8. CMD foreground = container stays alive
9. Layer caching = unchanged layers reused = fast builds
10. Same image in QA and Production — config differs via -e flags in deploy.sh
