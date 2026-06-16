A real end-to-end project you can build and own (points 3, 5, 6)
Here's a project that's realistic for your level, uses AWS, and you can build it yourself in the coming weeks while you prep for interviews. I'll call it the "3-Tier Microservices App with Full CI/CD on AWS":
The application: A simple app with 2-3 services — e.g., a Node.js/Python "users" API, a "orders" API, and a React/static frontend. You don't need to write complex code; even basic CRUD services are fine. The point is it's "microservices" (multiple independently deployable units).
What you build around it:

Each service gets a Dockerfile, built into images, pushed to AWS ECR.
Infrastructure (VPC, subnets, EC2 or EKS cluster, RDS for the database, S3 for static assets/frontend) provisioned via Terraform — this is your IaC story.
CI/CD with Jenkins or GitHub Actions: on every push, run tests, build Docker image, push to ECR, deploy to EKS/EC2.
Deploy to EKS (or start simpler with ECS/EC2 + docker-compose if Kubernetes feels heavy initially, then "graduate" to EKS).
Add Prometheus + Grafana for monitoring, and ship logs to CloudWatch or a basic ELK stack.
Use Ansible for any config management on EC2 instances (patching, installing agents).

This single project touches almost every topic in your 15-day plan, and gives you a legitimate story: "I built this end-to-end to learn the full lifecycle — here's the architecture, here's a problem I hit and how I solved it."
For your resume, a bullet might look like: "Designed and deployed a 3-tier microservices application on AWS using Docker, EKS, and Terraform; built CI/CD pipeline with Jenkins reducing deployment time from manual (~30 min) to automated (~5 min); configured Prometheus/Grafana for monitoring."

