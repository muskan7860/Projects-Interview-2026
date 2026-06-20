# MASTER PROJECT DOC — Retail Banking Platform (DevOps-Focused, 2.2 Years)

## 0. The Plain-Language Architecture (understand this FIRST, before professional wording)

**The mental model:**
- The application is just CODE (logic like "check balance," "transfer money") written by developers
- This code runs inside a Docker container — a sealed, portable package containing the code + everything it needs
- This container runs ON TOP of an EC2 instance (a virtual Linux server) — Docker is installed on the EC2, and the container runs inside Docker
- We run 3 EC2 instances, each running an IDENTICAL copy of the container — like 3 photocopies of the same ATM software, running independently
- All 3 copies are active and serving traffic SIMULTANEOUSLY — not "one main server + 2 backups sitting idle." The ALB spreads incoming requests across all 3 in parallel. "Healthy" only matters when one is broken — then ALB stops sending it traffic until it recovers.
- For any ONE individual request, the ALB sends it to exactly ONE of the 3 servers. But across many requests happening continuously, all 3 servers stay busy at the same time.
- None of the 3 containers store data themselves (stateless) — they're interchangeable. The ONLY thing they share is the database.
- The database CANNOT be "3 active copies" like the app servers, because data must have one single source of truth (if Riya's balance update went to one DB copy but her next request hit a different copy, she'd see stale data — unacceptable for banking). So instead: ONE primary database actively serves all read/write traffic, and ONE standby replica (RDS Multi-AZ) stays in sync silently in the background. If the primary fails, AWS automatically promotes the standby to become the new primary (failover) — app servers reconnect to the same database endpoint without needing to know a switch happened.

**The full request journey (the "walk me through it" answer):**
1. Customer opens browser, goes to the bank's website (e.g., bankname.com)
2. DNS converts that website name into the ALB's actual network address
3. Request arrives at the ALB (sitting in the public subnet)
4. ALB has been continuously health-checking all 3 app servers; it picks one HEALTHY server and forwards this specific request to it
5. That server's container (running the app code) processes the request — say, "get balance for customer X"
6. The container doesn't store data itself, so it opens a connection to the RDS SQL Server primary database and runs a query
7. Database returns the result to the app server
8. App server packages the response and sends it back, through the ALB, to the customer's browser
9. Customer sees their balance

**Public vs Private — the access logic:**
- ALB sits in the PUBLIC subnet — it's the only thing the internet can directly reach
- App servers and database sit in PRIVATE subnets — no direct internet access at all
- ALB can reach the app servers (allowed by security group rules) because that's the intended traffic path
- App servers can reach the database (allowed by security group rules) for the same reason
- The database NEVER talks directly to the ALB — it only ever talks to the app servers. Data flows back the same path it came: Database → App Server → ALB → Customer

---



"Our overall engineering org for this banking platform had around 35-40 people. Breaking that down:

- **Development team** (~15 people): wrote the actual banking application code (Java/Spring Boot based), split into 2-3 squads (e.g., Accounts squad, Payments squad)
- **QA team** (~6 people): functional and regression testing before releases
- **DevOps/Infra team** (~6 people): this is my team — responsible for CI/CD pipelines, infrastructure provisioning, deployments, monitoring, and production support
- **DBA team** (~3 people): managed database performance, backups, and DB-level changes (I coordinated with them often, given my own DBA background)
- **Security/Compliance team** (~2-3 people): reviewed IAM policies, security group changes, ran compliance audits (banking = heavy compliance)

**My team (DevOps, 6 people) breakdown:**
- 1 DevOps Lead/Architect (designed pipelines, infra decisions, handled production approvals)
- 2 Senior DevOps Engineers (owned Terraform/IaC, complex troubleshooting, mentored juniors)
- 3 Junior/Mid DevOps Engineers (**this was me and 2 peers**) — day-to-day pipeline maintenance, Docker builds, deployment execution, monitoring, smaller automation tasks

**How we communicated:**
- Daily standup (15 min) — what you did yesterday, what you're doing today, any blockers
- Tickets tracked in **Jira** (dev-facing work) and **ServiceNow** (incidents/infra requests)
- A dedicated **Slack/Teams channel** for deployment notifications and alerts (CloudWatch alarms posted here automatically)
- Weekly sync with the DBA team to review any DB-related tickets or upcoming schema changes affecting deployments
- For production releases, a **release call** (30-60 min) with Dev + QA + DevOps + a release manager, going through a checklist before approving the deploy"

---

## 2. Your Exact Role

"I was a Junior to Mid-level DevOps Engineer. I owned the CI/CD pipeline maintenance for our application, handled containerization of our services with Docker, supported daily deployments to QA/UAT, and was the go-to person on the team for database-related troubleshooting because of my prior DBA background. I also did basic infrastructure changes using Terraform under guidance from the senior engineers."

---

## 3. Architecture (Updated — Docker Added)

```
                                   INTERNET
                                       |
                                       |  (HTTPS - port 443)
                                       v
                    ===========================================
                    |              VPC (10.0.0.0/16)          |
                    |                                          |
                    |   PUBLIC SUBNET                          |
                    |   ----------------------------           |
                    |   |   ALB (Load Balancer)   |             |
                    |   ----------------------------           |
                    |   |   Bastion Host          |             |
                    |   ----------------------------           |
                    |              |                            |
                    |              | (port 8080)                |
                    |              v                            |
                    |   PRIVATE SUBNET - App Tier               |
                    |   --------------------------------------  |
                    |   | EC2 Host 1  | EC2 Host 2 | EC2 Host 3| |
                    |   |  [Docker]   |  [Docker]  |  [Docker] | |
                    |   |  banking-   |  banking-  |  banking- | |
                    |   |  app:tag    |  app:tag   |  app:tag  | |
                    |   --------------------------------------  |
                    |              |                            |
                    |              | (port 1433 - SQL Server)   |
                    |              v                            |
                    |   PRIVATE SUBNET - DB Tier                |
                    |   ----------------------------           |
                    |   |  RDS SQL Server (Primary) |          |
                    |   |  RDS SQL Server (Standby) |          |
                    |   ----------------------------           |
                    |                                          |
                    ===========================================

   SUPPORTING SYSTEMS (outside the VPC, but part of the pipeline):
   - GitHub (source code)
   - Jenkins Server (EC2, runs the pipeline)
   - Docker Hub / Amazon ECR (container image registry)
   - CloudWatch (monitoring/alarms)
   - Terraform (provisions the EC2s, security groups, etc. — state stored in S3)
```

**Key change from before:** the application doesn't run directly on Tomcat installed on the OS anymore — it runs **inside a Docker container** on each EC2 host. This is what makes deployments faster and more consistent across QA/UAT/Prod, and it's the centerpiece of your "I did real DevOps work" story.

---

## 4. Execution Flow — CI/CD Pipeline (Docker-based)

1. Developer pushes code to GitHub, opens a PR, gets it merged to `develop`
2. GitHub webhook triggers the **Jenkins pipeline** (Jenkinsfile, stored in the repo itself — "pipeline as code")
3. **Stage 1 — Checkout**: Jenkins pulls the latest code
4. **Stage 2 — Build & Unit Test**: Maven builds the Java app, runs unit tests (`mvn clean package`)
5. **Stage 3 — Docker Build**: Jenkins runs `docker build -t banking-app:<build-number> .` using a Dockerfile in the repo — this packages the app + its runtime (Tomcat/JRE) into a single image
6. **Stage 4 — Push to Registry**: Jenkins tags the image and pushes it to **Docker Hub / ECR** (`docker push`)
7. **Stage 5 — Deploy to QA**: Jenkins SSHs into the QA EC2 host and runs a deployment script: stop old container → `docker pull` the new image → `docker run` the new container → wait → hit health check endpoint
8. **Stage 6 — Notify**: Jenkins posts a Slack/Teams message — success or failure — to the team channel
9. **For Production**: same pipeline, but it pauses at a **manual approval gate** (Jenkins `input` step) — a senior engineer or release manager clicks "Approve" after reviewing QA sign-off

**Your specific contribution to this pipeline (what you can say you built):**
- Wrote the **Dockerfile** for the application (base image, copying the WAR file, exposing the port, entrypoint command)
- Wrote/maintained sections of the **Jenkinsfile** (the Docker build and push stages specifically)
- Wrote the **deployment shell script** used in Stage 5 (stop/pull/run/health-check logic)
- Set up the **Slack notification step** in the pipeline

---

## 5. Daily Tasks (DevOps-framed, 2.2 years)

**Morning (first 1-2 hours):**
- Check Slack channel / email for any overnight deployment failure notifications or CloudWatch alarms
- Review last night's scheduled Jenkins job (nightly QA deploy) — confirm it succeeded; if failed, open console log, identify the failing stage
- Quick SSH health check on app hosts: `docker ps` (confirm containers are running, check uptime/restarts), `df -h` (disk space — Docker images/logs can fill disk fast), `docker logs <container>` for recent errors

**Through the day:**
- Pick up Jira/ServiceNow tickets — typical examples:
  - "Add a new stage to the Jenkinsfile to scan the Docker image for vulnerabilities" (basic Trivy/Docker scan integration)
  - "Old Docker images are filling up disk on the build server — set up cleanup"
  - "New microservice needs a Dockerfile — base it on the existing one"
  - "Update Terraform to add a new security group rule allowing port X from Y"
  - "Increase the EC2 instance type for app-server-2 due to high memory usage" (modify Terraform, apply)
  - "Add a new developer's public key to the bastion for SSH access"
- Pair with a senior engineer on bigger changes (e.g., modifying the pipeline structure, Terraform module changes) — you write the first draft, they review

**Release windows (evenings/weekends, periodic):**
- Trigger/monitor the production deployment pipeline
- Watch `docker logs` post-deployment, confirm health check passes
- Be ready to **roll back** — re-deploy the previous image tag if something fails (`docker pull banking-app:<previous-tag>` and re-run)

**Ongoing/proactive work (the "I built things" part):**
- Wrote a cron job to prune unused Docker images weekly (`docker image prune`) after a few disk-full incidents
- Helped migrate 2-3 smaller internal tools/services to Docker as part of a broader containerization push the team was doing
- Wrote basic Terraform for a couple of small, well-defined changes (new SG rule, new EC2 instance) — always reviewed by a senior before `terraform apply`
