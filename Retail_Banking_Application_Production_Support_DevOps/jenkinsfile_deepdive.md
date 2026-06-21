# Jenkinsfile — Every Line Explained (Zero to Understanding)

---

## BEFORE WE READ THE FILE — understand what this file IS

Imagine you are a new manager at a factory.
Every morning, workers ask you: "What should we do today?"
You have to tell each worker, one by one, in order, what to do.

Now imagine you write all those instructions on ONE paper.
You stick that paper on the factory wall.
Now workers don't ask you — they just read the paper and follow it.

That paper = the Jenkinsfile.
The factory = Jenkins.
The workers = the stages (Checkout, Build, Test, Docker Build etc.)

This file sits INSIDE the GitHub repository, next to the application code.
When a developer pushes new code, Jenkins reads this file and follows
every instruction automatically — without any human doing anything manually.

---

## THE COMPLETE JENKINSFILE — line by line

```groovy
pipeline {
```
**What this means**: "Everything inside these curly braces { } is the
pipeline — the complete set of instructions Jenkins should follow."
Think of it as opening a book. Everything inside = the story Jenkins reads.

```groovy
    agent any
```
**What this means**: "Run this pipeline on ANY available Jenkins machine."
Jenkins can have multiple machines (called agents or slaves) for running
builds. "any" means: whichever one is free right now, use that.
In our setup we have one Jenkins server, so it always runs there.

**Behind the scenes**: Jenkins checks which agent is available,
assigns this build to it, and all commands in this pipeline run
on that machine.

---

```groovy
    environment {
        IMAGE_NAME    = "banking-app"
        IMAGE_TAG     = "${BUILD_NUMBER}"
        DOCKER_HUB_REPO = "yourteam/banking-app"
        APP_SERVER_1    = "10.0.1.10"
        APP_SERVER_2    = "10.0.1.11"
        APP_SERVER_3    = "10.0.1.12"
    }
```

**What this section is**: A list of VARIABLES — like shortcuts.
Instead of typing "banking-app" 10 times in the file, you define it
once here and use the variable name everywhere else.

**Why variables?** If the image name changes tomorrow, you change it
in ONE place here — the rest of the file updates automatically.
Without variables, you'd have to find and change it in 10 different places
and risk missing one.

**What is BUILD_NUMBER?**
Every time Jenkins runs a pipeline, it automatically assigns a number.
First run = 1, second run = 2, third run = 3, and so on.
Jenkins manages this counter automatically — you never set it yourself.
So `IMAGE_TAG = "${BUILD_NUMBER}"` means:
- First run: IMAGE_TAG = 1 → Docker image gets tag "banking-app:1"
- Second run: IMAGE_TAG = 2 → Docker image gets tag "banking-app:2"
- Forty-second run: IMAGE_TAG = 42 → Docker image gets tag "banking-app:42"

**Why is this important?**
Every deployment gets a unique version number.
If version 43 breaks production → you can go back to version 42 exactly.
If you always used the same tag ("latest"), you'd have no way to go back.

**What are APP_SERVER_1, 2, 3?**
These are the private IP addresses of your 3 EC2 app servers in AWS.
Jenkins needs to know WHERE to send the new deployment.
Private IPs mean these servers are inside the VPC — not internet-facing.
Jenkins (also inside the VPC or connected via VPN) can reach them directly.

---

```groovy
    stages {
```
**What this means**: "Here begins the list of steps, in order."
Everything inside this block = the actual work Jenkins does.

---

## STAGE 1: Checkout

```groovy
        stage('Checkout') {
            steps {
                git branch: 'develop',
                    url: 'https://github.com/yourteam/banking-app.git'
            }
        }
```

**What 'stage' means**: A named section of work.
`stage('Checkout')` = "this section of work is called Checkout."
The name appears in Jenkins UI as a visible box — you can see each
stage pass (green) or fail (red) separately.

**What happens behind the scenes**:
1. Jenkins goes to GitHub at the URL you provided
2. It finds the 'develop' branch
3. It downloads (git clone/pull) all the code files onto the Jenkins server
4. Now the Jenkins server has a local copy of the latest code

**Why 'develop' branch and not 'main'?**
In our team's workflow:
- Developers write features on their own branches
- They merge their features into 'develop' (the "work in progress" branch)
- 'develop' is for QA testing
- Only after QA approves does code go to 'main' (production)
So Jenkins listens to 'develop' to auto-deploy to QA environment.

**What triggers THIS stage to start?**
A GitHub webhook. When a developer merges code into 'develop',
GitHub sends a message to Jenkins: "Hey, new code is here."
Jenkins receives it and starts Stage 1 automatically.

---

## STAGE 2: Build

```groovy
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
```

**What 'sh' means**: "Run this as a shell command on the server."
sh = shell. Like you opening a terminal and typing a command.
Jenkins runs this command on the build server (agent).

**What is Maven (mvn)?**
Maven is a build tool for Java applications.
Think of it like a chef's assistant — you give it raw ingredients (Java code)
and it follows a recipe (pom.xml file in the repo) to produce the final dish.

**Breaking down 'mvn clean package -DskipTests'**:

`clean` = "delete everything from the previous build first."
Why? Old leftover files from last build could mix with new files
and cause confusing errors. Start fresh every time.
Behind the scenes: deletes the 'target/' folder on the Jenkins server.

`package` = "compile the code and package it into a WAR file."
What is WAR? Web Application Archive.
Think of it as a ZIP file containing:
- All the compiled Java code (bytecode, not human-readable)
- HTML/CSS/JavaScript files
- Configuration files
- Everything the app needs to run — all in one single file

Behind the scenes:
1. Maven reads pom.xml (the recipe file) in the repo
2. Downloads any missing libraries the app needs
3. Compiles .java files → .class files (machine-readable)
4. Packages everything → target/banking-app.war

`-DskipTests` = "don't run tests in this stage."
We run tests separately in Stage 3.
Why separate? If build AND tests are combined and something fails,
you can't tell: "did the code fail to compile, or did a test fail?"
Keeping them separate gives a clear answer.

**After this stage**: File `target/banking-app.war` exists on Jenkins server.
This is the complete, compiled, packaged banking application.

---

## STAGE 3: Test

```groovy
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
```

**What happens**: Maven runs all unit tests that developers wrote.
Unit tests = small automated checks. Examples:
- "Does the login function reject wrong passwords?"
- "Does transfer money correctly deduct from sender and add to receiver?"
- "Does balance check return the right number?"

Developers write these tests alongside the application code.
There might be 50, 100, or 500 of these tests.
Maven runs ALL of them automatically.

**Behind the scenes**:
1. Maven finds all test files (files ending in Test.java)
2. Runs each test one by one
3. Tracks which pass and which fail
4. At the end: shows a report (X passed, Y failed)

**What if even ONE test fails?**
Jenkins marks Stage 3 as FAILED (red).
The pipeline STOPS completely.
Stages 4, 5, 6, 7, 8 DO NOT RUN.
The broken code never reaches deployment.
Jenkins sends a failure notification to the team.
A developer must fix the failing test before the pipeline can proceed.

**This is the most important safety gate in the entire pipeline.**
It's what ensures: broken code never reaches customers.

---

## STAGE 4: Docker Build

```groovy
        stage('Docker Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }
```

**What this command does**: Creates a Docker image.
`docker build` = "build a Docker image"
`-t` = "tag it with this name" (t = tag)
`${IMAGE_NAME}:${IMAGE_TAG}` = "banking-app:42" (using variables from top)
`.` = "look for the Dockerfile in the CURRENT folder"

**What is a Docker image?**
Remember our shipping container analogy?
A Docker image is the BLUEPRINT of that container.
It contains:
- A base Linux operating system (lightweight)
- Java runtime (JRE 17)
- Tomcat web server
- Our banking-app.war file (copied in during build)

The image is READ-ONLY. It's like a template or a mold.
When you RUN an image, it creates a CONTAINER — the live, running instance.
Image = recipe. Container = the actual cake made from the recipe.

**Behind the scenes of 'docker build'**:
1. Docker reads the Dockerfile (explained in separate file)
2. Follows each instruction in Dockerfile step by step
3. Creates LAYERS — each instruction creates one layer
4. Combines all layers into one image
5. Tags it as "banking-app:42"
6. Stores it locally on the Jenkins server

**Why "banking-app:42" specifically?**
The colon separates name from tag.
Name = banking-app (what the application is)
Tag = 42 (which version/build this is)
This image now has a unique identity — no other build has this exact tag.

---

## STAGE 5: Docker Push

```groovy
        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_HUB_REPO}:${IMAGE_TAG}
                        docker push ${DOCKER_HUB_REPO}:${IMAGE_TAG}
                    """
                }
            }
        }
```

**The problem this stage solves**:
After Stage 4, the image "banking-app:42" exists ONLY on the Jenkins server.
But our 3 app servers (10.0.1.10, 10.0.1.11, 10.0.1.12) need to get it too.
We need a central storage location — like a warehouse — where Jenkins uploads
the image and any server can download it from.
That warehouse = Docker Hub.

**What is Docker Hub?**
Docker Hub is a cloud service (like GitHub but for Docker images).
You push images to it. Servers pull images from it.
Free for public repos, paid for private.
In real companies, you'd use a private registry like AWS ECR
(Elastic Container Registry) for security — no public access.

**What is 'withCredentials'?**
This is Jenkins' way of safely using passwords.
`credentialsId: 'dockerhub-credentials'` = "look up the stored secret
called 'dockerhub-credentials' in Jenkins' secure vault."

A Jenkins admin previously saved the Docker Hub username and password
in Jenkins under this name — encrypted, never visible in plain text.
`withCredentials` temporarily makes them available as environment
variables (DOCKER_USER and DOCKER_PASS) just for the commands inside
this block. After the block ends, they disappear from memory.

**WHY NOT just type the password directly in the Jenkinsfile?**
The Jenkinsfile lives in GitHub. If someone can read the repo
(and in many companies, all developers can), they'd see the password.
Credentials manager keeps passwords hidden even from people who
can read the Jenkinsfile.

**Breaking down the three commands inside**:

`echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin`
= Log into Docker Hub using the stored username and password.
`--password-stdin` = take the password from input (not typed directly)
= more secure than `-p yourpassword` which shows in process logs.

`docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_HUB_REPO}:${IMAGE_TAG}`
= Give the image a second name that includes the Docker Hub account.
Before: "banking-app:42" (local name)
After tag: "yourteam/banking-app:42" (Docker Hub name, includes account)
Why needed? Docker Hub requires images to be named with your account prefix.

`docker push ${DOCKER_HUB_REPO}:${IMAGE_TAG}`
= Upload the image to Docker Hub.
Behind the scenes: Docker uploads each layer of the image.
If a layer was already uploaded from a previous build (unchanged),
Docker skips it — only uploads NEW or changed layers. Efficient.

**After this stage**: Image "yourteam/banking-app:42" exists on Docker Hub.
Any server with internet/network access can now pull this exact version.

---

## STAGE 6: Deploy to QA

```groovy
        stage('Deploy to QA') {
            steps {
                sh """
                    ssh ubuntu@${APP_SERVER_1} \
                        'bash /opt/scripts/deploy.sh ${IMAGE_TAG}'
                    ssh ubuntu@${APP_SERVER_2} \
                        'bash /opt/scripts/deploy.sh ${IMAGE_TAG}'
                    ssh ubuntu@${APP_SERVER_3} \
                        'bash /opt/scripts/deploy.sh ${IMAGE_TAG}'
                """
            }
        }
```

**What 'ssh' means here**:
SSH = Secure Shell = a way to log into another computer remotely
and run commands on it, from another machine.

`ssh ubuntu@${APP_SERVER_1}` = "log into the server at IP 10.0.1.10
as user 'ubuntu' (the default Linux user on Ubuntu EC2 instances)"

`'bash /opt/scripts/deploy.sh ${IMAGE_TAG}'`
= "once logged in, run this script, and pass it the build number (42)"

**Behind the scenes**:
1. Jenkins (on Jenkins server) opens an SSH connection to 10.0.1.10
2. Authenticates using SSH key (stored in Jenkins credentials — not a password)
3. Runs the deploy.sh script on THAT server (not on Jenkins server)
4. deploy.sh runs, swaps containers, does health check (explained separately)
5. If deploy.sh exits with 0 = success, Jenkins moves to next server
6. If deploy.sh exits with 1 = failure, Jenkins marks stage FAILED

**Why SSH key and not password?**
SSH keys are more secure and can't be brute-forced.
Jenkins has the private key stored in its credentials vault.
The app servers have the corresponding public key in
`/home/ubuntu/.ssh/authorized_keys`.
This way Jenkins can log in without a password being transmitted.

**Why do we deploy to ALL 3 servers?**
Because all 3 are serving live traffic (remember — ALB distributes
requests across all 3 simultaneously). If we only updated 1 server,
customers would get different versions of the app depending on which
server the ALB sent them to. Version mismatch = inconsistent behaviour.
All 3 must run the same version at all times.

**Important note about order**:
We deploy one server at a time (1, then 2, then 3).
During deployment on server 1, servers 2 and 3 are still running
the OLD version and serving customers normally.
The ALB's health check detects server 1 is temporarily down
(container stopped) and automatically routes all traffic to 2 and 3.
When server 1 comes back up with new version, ALB includes it again.
Then server 2 gets updated, etc.
This is called a ROLLING DEPLOYMENT — zero downtime for customers.

---

## STAGE 7: Approval Gate

```groovy
        stage('Approval for Production') {
            steps {
                input message: 'QA verified? Approve to deploy to Production?',
                      ok: 'Deploy to Production',
                      submitter: 'senior-engineer,release-manager'
            }
        }
```

**What happens**: Pipeline FREEZES here. Completely stops.
Jenkins sends a notification (Slack/email) to the team:
"Pipeline is waiting for approval to deploy to production."

A senior engineer or release manager:
1. Checks QA test results (manual and automated)
2. Confirms no known issues
3. Opens Jenkins in browser
4. Clicks "Deploy to Production" button

Only then does the pipeline continue to Stage 8.

**What if nobody approves?**
You can configure a timeout (e.g., 24 hours).
If nobody approves within 24 hours, the pipeline auto-cancels.
The build stays deployed in QA, not in production.
Team must restart the pipeline next release window.

**'submitter'** = only THESE named people can approve.
A junior engineer cannot accidentally click approve.
Access control built directly into the pipeline.

**Why does this exist for banking specifically?**
Regulatory compliance — banking applications in India (RBI guidelines)
and globally require human authorization before production changes.
An automated pipeline that deploys to production without human approval
would fail a compliance audit.

---

## STAGE 8: Deploy to Production

```groovy
        stage('Deploy to Production') {
            steps {
                sh """
                    ssh ubuntu@${APP_SERVER_1} \
                        'bash /opt/scripts/deploy.sh ${IMAGE_TAG}'
                    ssh ubuntu@${APP_SERVER_2} \
                        'bash /opt/scripts/deploy.sh ${IMAGE_TAG}'
                    ssh ubuntu@${APP_SERVER_3} \
                        'bash /opt/scripts/deploy.sh ${IMAGE_TAG}'
                """
            }
        }
```

**Identical to Stage 6** — same script, same process, same servers.
(In real setup, production server IPs would be different from QA IPs.
Here simplified for explanation — in reality you'd have separate
environment variables for QA_SERVER_1 and PROD_SERVER_1.)

**Why keep it identical to QA deploy?**
If QA and Production use different deployment methods,
you introduce a new unknown: "what if the deployment PROCESS itself
works differently between environments?"
By keeping them identical, the ONLY variable is the environment.
If it deployed successfully in QA using this exact method,
it will deploy successfully in Production using the same method.

---

## POST BLOCK

```groovy
    post {
        success {
            echo "✅ Build ${IMAGE_TAG} deployed successfully to Production"
        }
        failure {
            echo "❌ Pipeline failed — check console output"
        }
    }
```

**What 'post' means**: "After ALL stages finish — whether they succeeded
or failed — run these actions."

`success { }` = runs ONLY if every single stage passed
`failure { }` = runs if ANY stage failed

**In real production setup**, instead of just `echo`:
- Success: send Slack message to #deployments channel
  "✅ banking-app:42 deployed to production successfully"
- Failure: send alert to #alerts channel + page on-call engineer
  "❌ Pipeline FAILED at [stage name] — build 42"

**Why is this important?**
Without notifications, nobody knows a build failed unless they
manually check Jenkins. In a real team with multiple pipelines
running, you CANNOT manually check every build.
Notifications mean the right person is informed immediately.

---

## THE COMPLETE FLOW — one picture

```
Developer merges code to 'develop' branch on GitHub
                    ↓
GitHub Webhook pings Jenkins: "new code!"
                    ↓
Stage 1 CHECKOUT: Jenkins downloads latest code from GitHub
                    ↓
Stage 2 BUILD: Maven compiles code → banking-app.war file created
                    ↓
Stage 3 TEST: Maven runs all unit tests
              ↓ (if ANY test fails — STOP. Nobody deploys.)
              ↓ (if ALL tests pass — continue)
Stage 4 DOCKER BUILD: Docker reads Dockerfile, creates image banking-app:42
                    ↓
Stage 5 DOCKER PUSH: Image uploaded to Docker Hub warehouse
                    ↓
Stage 6 DEPLOY QA: Jenkins SSHs into 3 servers, runs deploy.sh 42
                   (old container stopped, new container started, health check)
                    ↓
Stage 7 APPROVAL: Pipeline PAUSES. Senior engineer reviews QA. Clicks Approve.
                    ↓
Stage 8 DEPLOY PROD: Same as QA deploy, on production servers
                    ↓
POST: Team notified ✅ or ❌
```

---

## KEY INTERVIEW CONNECTIONS TO REMEMBER

- Jenkinsfile lives IN the GitHub repo → "pipeline as code"
- BUILD_NUMBER → unique tag → rollback capability
- withCredentials → never hardcode passwords
- SSH into servers → Jenkins needs SSH key in credentials vault
- Rolling deployment → no downtime (1 server updates while others serve)
- Manual approval → compliance requirement for banking
- Post block → team notification without manual checking
- If TEST stage fails → NOTHING deploys → safety gate
