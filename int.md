Int

Docker:

__Q: What is Dockerfile? Explain key instructions.__

A: Dockerfile is a text file with instructions to build a Docker image.

Key instructions:

- FROM: Base image
- RUN: Execute commands during build
- COPY/ADD: Copy files to image
- WORKDIR: Set working directory
- EXPOSE: Document which ports to expose
- CMD/ENTRYPOINT: Define default command to run


# CMD - can be overridden
CMD ["python", "app.py"]
# Running: docker run image bash (overrides CMD)

# ENTRYPOINT - cannot be easily overridden
ENTRYPOINT ["python", "app.py"]
# Running: docker run image (always runs python app.py)

__Q: What are Docker volumes? When do you use them?__

A: Volumes persist data beyond container lifecycle.

Use cases:

- Database data persistence
- Sharing data between containers
- Mounting config files
- Development with hot reload

Types:

- Named volumes: docker volume create myvol
- Bind mounts: -v /host/path:/container/path
- Anonymous volumes: -v /container/path


__Q: How do Docker networks work? Explain types.__

A:

- Bridge (default): Containers on same host communicate
- Host: Container shares host network (no isolation)
- None: No network access
- Overlay: Multi-host communication (Swarm/Kubernetes)
- Macvlan: Direct physical network connection


### 3. Advanced Level (Production Scenarios)

__Q: A container keeps crashing. How do you troubleshoot?__

A:

1. Check container logs: docker logs container_id
2. Inspect container: docker inspect container_id
3. Check resource usage: docker stats
4. Look at exit code: docker ps -a (shows exit codes)
5. Run interactively for debugging: docker run -it image /bin/bash
6. Check host resources: df -h, free -m, top
7. Review application logs inside container

Common exit codes:

- 0: Clean exit
- 1: Application error
- 137: OOMKilled (memory exceeded)
- 143: SIGTERM received

---

__Q: How do you handle secrets in Docker?__

A: Never store secrets in Dockerfile or images!

Best practices:

1. Docker Secrets (Swarm mode)
2. Environment variables at runtime: -e DB_PASSWORD=secret
3. Mount secret files as volumes
4. Use external secret managers (Vault, AWS Secrets Manager)
5. Docker Compose secrets for local dev



__Q: Container is running but application is not accessible. Debug steps?__

A:

1. Check if port is mapped: docker port container_id
2. Verify application is listening: docker exec container netstat -tlnp
3. Check firewall rules on host
4. Test from inside container: docker exec container curl localhost:port
5. Check application logs for startup errors
6. Verify HEALTHCHECK status: docker inspect --format='{{.State.Health}}' container
7. Check network connectivity: docker network inspect


__: Explain Docker layer caching. How do you optimize it?__

A: Docker caches each layer. Layers are cached based on instruction and content.

Optimization strategies:

1. Order Dockerfile instructions by change frequency
2. Copy package files first, then install dependencies
3. Copy source code last (changes most frequently)
4. Use specific base image tags (not latest)

Example - Optimized order:

```dockerfile
FROM node:18-alpine
WORKDIR /app

# Rarely changes - copy first
COPY package*.json ./
RUN npm ci --only=production

# Changes often - copy last
COPY . .
CMD ["node", "index.js"]
```

---

__Q: How do you secure Docker containers in production?__

A:

1. Run as non-root user:

   ```dockerfile
   RUN adduser -D appuser
   USER appuser
   ```

2. Use minimal base images (alpine, distroless)

3. Scan images for vulnerabilities (Trivy, Snyk, Clair)

4. Set resource limits:

   ```bash
   docker run --memory=512m --cpus=1.0 image
   ```

5. Use read-only filesystem where possible:

   ```bash
   docker run --read-only --tmpfs /tmp image
   ```

6. Drop unnecessary capabilities:

   ```bash
   docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE image
   ```

7. Keep images updated, use specific tags

8. Don't run privileged containers unless absolutely necessary

---

__Q: Docker Compose vs Kubernetes - When to use what?__

A: Docker Compose:

- Single host
- Development/local testing
- Simple applications
- Up to ~10-20 containers

Kubernetes:

- Multi-host/cluster
- Production workloads
- Complex microservices
- Auto-scaling, self-healing
- Hundreds/thousands of containers

Migration path: Docker Compose for dev -> Kubernetes for prod
——————————————————————————————————————————————————————————————————————————————————————————————————
Terraform:

——————————————————————————————————————————————————————————————————————————————————————————————————
Kubernetes:
1.Walk me through how you upgraded an Amazon EKS cluster from Kubernetes 1.32 to 1.34 without downtime.

Here’s how I would explain it in an interview 👇

Yeah, sure. In one of my projects, we had a customer-facing application running as microservices on Amazon EKS, and we had a requirement to upgrade Kubernetes from 1.32 to 1.34.
Since it was Production, we couldn't afford application downtime, so we followed a proper upgrade runbook.

1️⃣ Pre-checks
Before touching the cluster, I checked:
• EKS Upgrade Insights
• Deprecated Kubernetes APIs
• Helm chart and application compatibility
• VPC CNI, CoreDNS and kube-proxy
• AWS Load Balancer Controller
• PDBs and replica counts
• Monitoring components

Once the pre-checks were clean, we tested the complete upgrade in a lower environment first.

We never started directly with Production.

2️⃣ Upgrade one version at a time
Since EKS upgrades are performed one minor version at a time, we followed:
1.32 → 1.33 → Validate → 1.34
For each version, I first upgraded the EKS control plane.
Once the control plane was healthy, I validated the cluster and then upgraded the required EKS add-ons.

3️⃣ Replace the worker nodes
For worker nodes, we didn't immediately terminate the existing node group.
We created a new managed node group with an EKS-optimized AMI compatible with the new Kubernetes version.
Once the new nodes joined the cluster and showed Ready, we started moving the workloads.
Cordon old node → Drain one node at a time → Pods move to new nodes
We didn't drain everything together.
We moved the workloads gradually to maintain application availability.

4️⃣ How did we avoid downtime?
Our critical microservices had:
✅ Multiple replicas
✅ PodDisruptionBudgets
✅ Readiness probes
✅ Multi-AZ workload distribution
So even when one node was being drained, healthy Pods on other nodes continued serving customer traffic.

5️⃣ Monitor throughout the upgrade
Using CloudWatch and Prometheus/Grafana, I continuously monitored:
• Node health
• Pod restarts
• Pending Pods
• CPU and memory
• ALB target health
• Application latency
• 5xx errors

6️⃣ Validate before removing anything
Once all workloads were running on the new node group, we didn't immediately remove the old one.
We first performed smoke testing and validated critical application flows:
Customer Login → Account Information → API Connectivity → Transaction Processing
Once everything was stable, we removed the old node group.
Then we repeated the same process for:
1.33 → 1.34

💡 The key interview point:
Zero downtime wasn't achieved just because we carefully upgraded EKS.

It was possible because the application was already designed for high availability with multiple replicas, PDBs, readiness probes, Multi-AZ deployment, and controlled node draining.

That’s how I would explain a real Production EKS upgrade in an interview.

———————————————————————————————————————————————————————————————————————————————————————————————————————
Jenkins

2.✅ AWS DEVOPS SCENARIO BASED INTERVIEW QUESTIONS & ANSWERS

1. You need to implement a CI/CD pipeline for a Java application stored in GitHub and deploy it on EC2. How will you do it?

Ans:

Use AWS CodePipeline for CI/CD orchestration.

Use CodeBuild to build the code, run unit tests and create artifact.

Store artifact (JAR) in S3.

Use CodeDeploy to deploy the artifact to EC2 instances.

Use CodeDeploy Agent installed on EC2 for deployment.

Monitor the pipeline using CloudWatch.

2. Your application is containerized using Docker. How will you automate the build, push and deployment of the image to ECS?

Ans:

Use CodePipeline for CI/CD.

Use CodeBuild to build Docker image.

Push the image to Amazon ECR.

Use ECS (Fargate or EC2 Launch Type) for deployment.

Update the ECS service with the new task definition.

Enable CloudWatch for logs and monitoring.

3. How will you manage infrastructure as code (IaC) on AWS?

Ans:

Use AWS CloudFormation or Terraform to define infrastructure.

Store templates in Git.

Use CodePipeline to validate and deploy changes.

Use different environments (dev, stage, prod) with parameters. Enable versioning and review changes through pull requests.

4. Your application is facing high latency. How will you monitor and troubleshoot it on AWS?

Ans:

Use CloudWatch Metrics, Alarms and Dashboards to monitor resources.

Enable CloudWatch Logs for application logs. Use AWS X-Ray for distributed tracing.

Check EC2 metrics (CPU, Memory, Network, Disk). Use CloudWatch Logs Insights to analyze logs.

Identify bottlenecks and optimize the application or infrastructure.

5. You need to deploy a blue/green deployment for your application.

Ans:

How will you achieve it on AWS?

Use CodeDeploy with Blue/Green deployment strategy. Create two environments (Blue & Green) behind an ALB.

Deploy new version to Green environment.

Run health checks and automated tests.

Shift traffic from Blue to Green using ALB.

Rollback automatically if issues occur.

6. You want to store and manage secrets (DB password, API keys) securely in your DevOps pipeline. How will you do it?

Ans:

Use AWS Secrets Manager or Systems Manager Parameter Store.

Store secrets securely with encryption.

Access secrets in CodeBuild/EC2/ECS using IAM roles. Never hardcode secrets in code or pipeline.

7.how u will implement centralised logging d alerting for was infra?8.your app need to be highly available and fault tolerant how will you design the infra	on was?

———————————————————————————————————————————————————————————————————————————————————————————————————————

Kubernetrs
📍1. What is Kubernetes?

Ans: Kubernetes is an open-source container orchestration platform that automates deployment, scaling, and management of containerized applications.

📍2. What is a Pod in Kubernetes?

Ans: A Pod is the smallest deployable unit in Kubernetes. It can contain one or more containers that share the same network and storage.

📍3. What is the difference between Deployment and StatefulSet?

Ans: Deployment is used for stateless applications and provides replica management. StatefulSet is used for stateful applications and provides stable network identity and persistent storage.

📍4. How does Kubernetes handle scaling?

Ans: Kubernetes supports horizontal scaling through ReplicaSets and Deployments. You can manually scale using kubectl or automatically using Horizontal Pod Autoscaler (HPA).

📍5. What is a Service in Kubernetes?

Ans: A Service is an abstraction that defines a logical set of Pods and a policy to access them. It provides stable IP and DNS name for accessing the Pods.
6.ISTIO
7.Application is experiencing high traffic sometimes it becomes slow how do scale and highly available?
1. You need to implement a CI/CD pipeline for a Java application stored in GitHub and deploy it on EC2. How will you do it?

Ans:

Use AWS CodePipeline for CI/CD orchestration.

Use CodeBuild to build the code, run unit tests and create artifact.

Store artifact (JAR) in S3.

Use CodeDeploy to deploy the artifact to EC2 instances.

Use CodeDeploy Agent installed on EC2 for deployment.

Monitor the pipeline using CloudWatch.

2. Your application is containerized using Docker. How will you automate the build, push and deployment of the image to ECS?

Ans:

Use CodePipeline for CI/CD.

Use CodeBuild to build Docker image.

Push the image to Amazon ECR.

Use ECS (Fargate or EC2 Launch Type) for deployment.

Update the ECS service with the new task definition.

Enable CloudWatch for logs and monitoring.

3. How will you manage infrastructure as code (IaC) on AWS?

Ans:

Use AWS CloudFormation or Terraform to define infrastructure.

Store templates in Git.

Use CodePipeline to validate and deploy changes.

Use different environments (dev, stage, prod) with parameters. Enable versioning and review changes through pull requests.

4. Your application is facing high latency. How will you monitor and troubleshoot it on AWS?

Ans:

Use CloudWatch Metrics, Alarms and Dashboards to monitor resources.

Enable CloudWatch Logs for application logs. Use AWS X-Ray for distributed tracing.

Check EC2 metrics (CPU, Memory, Network, Disk). Use CloudWatch Logs Insights to analyze logs.

Identify bottlenecks and optimize the application or infrastructure.

5. You need to deploy a blue/green deployment for your application.

Ans:

How will you achieve it on AWS?

Use CodeDeploy with Blue/Green deployment strategy. Create two environments (Blue & Green) behind an ALB.

Deploy new version to Green environment.

Run health checks and automated tests.

Shift traffic from Blue to Green using ALB.

Rollback automatically if issues occur.

6. You want to store and manage secrets (DB password, API keys) securely in your DevOps pipeline. How will you do it?

Ans:

Use AWS Secrets Manager or Systems Manager Parameter Store.

Store secrets securely with encryption.

Access secrets in CodeBuild/EC2/ECS using IAM roles. Never hardcode secrets in code or pipeline.

Here are the questions, section-wise 👇
🔹 1. Git & Branching Strategy
* GitFlow vs Trunk-Based — when would you choose each?
* In an application having 50+ microservices, how would you manage branching and independent deployments?
🔹 2. CI/CD Pipeline
* Design an end-to-end CI/CD flow for 50+ microservices.
* How many YAML pipelines would you maintain for 50+ microservices?
* Would you create one pipeline per microservice or a centralized pipeline?
* How would you avoid rebuilding all 50 services when only one microservice changes?
* How would you promote the same artifact from Dev → QA → Production?
* Where would you implement security scanning in the pipeline?
* How would you handle rollback?
🔹 3. Microservices Architecture
* Explain a 3-tier microservices architecture.
* Explain the Presentation, Application and Data tiers.
* Why are we using microservices instead of a monolith?
* What happens when one microservice is changed in an application containing 50+ services?
* How would you handle service-to-service communication?
* How would you reduce the impact/blast radius of a failed microservice?
🔹 4. API Gateway
* What is an API Gateway?
* Why do we need an API Gateway in microservices?
* How does API Gateway route traffic to different microservices?
* Where would you implement authentication, rate limiting and security?Why do we need an API Gateway in microservices?
* How does API Gateway route traffic to different microservices?
* Where would you implement authentication, rate limiting and security?
* API Gateway vs Ingress — what is the difference?
🔹 5. Redis
* What is Redis and why would you use it?
* Redis vs traditional database?
🔹 6. Azure Front Door — Active/Passive AKS
* Suppose we have two AKS sites: Primary and DR/Passive.
* How would you send traffic to the Passive AKS site during a Primary site failure?
* How would Azure Front Door detect that the Primary site is unhealthy?
* How would you configure Origin Groups and priority-based routing?
* What health endpoint would you configure?
* After the Primary site recovers, how would you perform failback?
* Does Front Door failover automatically solve database failover?
 🎯 What I felt the interviewer was actually evaluating
The interview was not just about **"Do you know Kubernetes/Terraform/Azure?"
The real evaluation was:
Can you design?
Can you automate?
Can you troubleshoot?
Can you handle production failures?
Can you explain trade-offs?
Can you think like a Lead DevOps Engineer?

📌 Questions asked:

1️⃣ Self Introduction
• Tell me about yourself.

2️⃣ Terraform – Deployment
• How have you used Terraform for deployment in your project?

3️⃣ Terraform – Folder Structure & Modules
• How did you structure your Terraform folders?
• How did you use Terraform modules in your project?

4️⃣ VPC & Subnets
• How did you implement a VPC with public and private subnets?
• How do you identify whether a subnet is public or private?

5️⃣ VPC Peering
• What is VPC Peering?
• How have you implemented VPC Peering in your project?

6️⃣ Amazon EKS – Project Experience
• How have you used Amazon EKS in your project?

7️⃣ EKS Cluster Upgrade
• How do you perform an EKS cluster upgrade?
• What is the process for upgrading an EKS cluster?

8️⃣ Terraform Remote Backend
• Which remote backend have you used with Terraform?
• How did you configure the Terraform remote backend?

9️⃣ S3 with Terraform
• How did you configure S3 for Terraform remote state?
• How did you define the S3 bucket in Terraform?


20 Crazy System Design Questions That Can Break a Senior Interview

These aren’t “design Twitter” questions.

These are the questions that force you to think beyond the architecture diagram.

1. Your system is 99.99% available, but customers still complain about downtime. How is that possible?
2. Redis is faster than your database. Why could removing Redis actually make your system more reliable?
3. Kafka guarantees message durability. Can you still lose a message?
4. Your database has plenty of CPU and memory, but response time suddenly jumps from 50ms to 5 seconds. What could be happening?
5. You have exactly-once processing. Why can users still see duplicate payments?
6. Your load balancer is working perfectly, but one server receives 80% of the traffic. Why?
7. You increased your service instances from 10 to 100, but throughput barely changed. What’s your bottleneck?
8. Your cache hit rate is 99%. Why can your database still be overloaded?
9. A request succeeds, but the client never receives the response and retries it. How do you prevent duplicate operations?
10. Your database replication lag is 30 seconds. Should you stop sending traffic to replicas?
11. Your system has no single point of failure, yet the entire system goes down. What could cause it?
12. You added retries to make your system more resilient, and suddenly the outage became 10x worse. Why?
13. Two services each report “success,” but the final business operation failed. How would you recover?
14. Your distributed lock expires while the first process is still working. What happens next?
15. You increased Kafka partitions and performance became worse. Why could that happen?
16. Your API latency is excellent at the 95th percentile but terrible at p99. What does that tell you?
17. You scaled horizontally, but your application became slower. What hidden shared resource could be limiting you?
18. Your database is strongly consistent, but users still see stale data. How?
19. Everything works perfectly in testing, but production randomly fails once every few hours. How would you design the system to find the problem?
20. Your architecture has zero downtime deployments — yet deployment still causes customer-visible failures. How?

The real System Design interview starts when the interviewer asks:

“Okay… but what happens if THAT fails?”

Don’t just learn how to draw architecture.

Learn how to challenge your own design.

System Design Interview Bundle:


Sharing my recent Infosys AWS Engineer interview experience to help others preparing for similar roles.

Below are some of the questions discussed during the interview:

AWS, Kubernetes & Terraform — Interview Questions

1. Introduce yourself.

2. What projects have you worked on?

3. What AWS services have you worked with?

4. Describe the recent project/work you were involved in.

5. What challenges did you encounter in your recent project, and how did you resolve them?

6. What challenges do you typically encounter during a Kubernetes upgrade?

7. What checks do you perform after a Kubernetes upgrade?

8. What was the Kubernetes version before the upgrade, and which version did you upgrade to?

9. What checks do you perform after the upgrade to ensure everything is working correctly?

10. What is the difference between managed and self-managed nodes?

11. What is the CIDR range of your subnet?

12. How many route tables can a subnet be associated with?

13. If you need to make changes to a route table, would you modify the existing one or create a new one? Why?

14. What would you do if the existing route table does not allow the required modification?

15. Have you ever created a Kubernetes cluster? What steps would you consider when creating one?

16. How would you configure autoscaling policies for a Kubernetes cluster?

17. What are the different ways you can create a Kubernetes cluster?

18. How many Kubernetes clusters are you currently managing?

19. How do you handle conflicts when multiple people are trying to modify infrastructure using Terraform?


Aug Shift Allowance : 

Emp ID	Emp Name	Account/Project Name	Date	Shift Type	Company Transportation Opted (Yes/No)	Work Mode (Office/WFH)	Amount Per Day (₹)	Month	Billable to Client
AIPL15449	Bethireddy Ravindra Kumar	Samsung	1 Aug 2026	Morning Shift (Starts anytime between 05:00 and 06:59 )	Yes	WFO	350	Aug 26	Yes
AIPL15449	Bethireddy Ravindra Kumar	Samsung	2 Aug 2026	Morning Shift (Starts anytime between 05:00 and 06:59 )	Yes	WFO	350	Aug 26	Yes
AIPL15449	Bethireddy Ravindra Kumar	Samsung	10 Aug 2026	Morning Shift (Starts anytime between 05:00 and 06:59 )	Yes	WFO	350	Aug 26	Yes
AIPL15449	Bethireddy Ravindra Kumar	Samsung	24 Aug 2026	Morning Shift (Starts anytime between 05:00 and 06:59 )	Yes	WFO	350	Aug 26	Yes
AIPL15449	Bethireddy Ravindra Kumar	Samsung	6 Aug 2026	Afternoon Shift (Starts anytime between 13:00 and 15:59 )	Yes	WFO	250	Aug 26	Yes
AIPL15449	Bethireddy Ravindra Kumar	Samsung	7 Aug 2026	Afternoon Shift (Starts anytime between 13:00 and 15:59 )	Yes	WFO	250	Aug 26	Yes
AIPL15449	Bethireddy Ravindra Kumar	Samsung	8 Aug 2026	Afternoon Shift (Starts anytime between 13:00 and 15:59 )	Yes	WFO	250	Aug 26	Yes
AIPL15449	Bethireddy Ravindra Kumar	Samsung	9 Aug 2026	Afternoon Shift (Starts anytime between 13:00 and 15:59 )	Yes	WFO	250	Aug 26	Yes
AIPL15449	Bethireddy Ravindra Kumar	Samsung	13 Aug 2026	Afternoon Shift (Starts anytime between 13:00 and 15:59 )	Yes	WFO	250	Aug 26	Yes
AIPL15449	Bethireddy Ravindra Kumar	Samsung	14 Aug 2026	Afternoon Shift (Starts anytime between 13:00 and 15:59 )	Yes	WFO	250	Aug 26	Yes
AIPL15449	Bethireddy Ravindra Kumar	Samsung	15 Aug 2026	Afternoon Shift (Starts anytime between 13:00 and 15:59 )	No	WFH	250	Aug 26	Yes
AIPL15449	Bethireddy Ravindra Kumar	Samsung	21 Aug 2026	Afternoon Shift (Starts anytime between 13:00 and 15:59 )	Yes	WFO	250	Aug 26	Yes
AIPL15449	Bethireddy Ravindra Kumar	Samsung	22 Aug 2026	Afternoon Shift (Starts anytime between 13:00 and 15:59 )	Yes	WFO	250	Aug 26	Yes
AIPL15449	Bethireddy Ravindra Kumar	Samsung	23 Aug 2026	Afternoon Shift (Starts anytime between 13:00 and 15:59 )	Yes	WFO	250	Aug 26	Yes
AIPL15449	Bethireddy Ravindra Kumar	Samsung	27 Aug 2026	Afternoon Shift (Starts anytime between 13:00 and 15:59 )	Yes	WFO	250	Aug 26	Yes
AIPL15449	Bethireddy Ravindra Kumar	Samsung	28 Aug 2026	Afternoon Shift (Starts anytime between 13:00 and 15:59 )	Yes	WFO	250	Aug 26	Yes
AIPL15449	Bethireddy Ravindra Kumar	Samsung	29 Aug 2026	Afternoon Shift (Starts anytime between 13:00 and 15:59 )	Yes	WFO	250	Aug 26	Yes
AIPL15449	Bethireddy Ravindra Kumar	Samsung	30 Aug 2026	Afternoon Shift (Starts anytime between 13:00 and 15:59 )	Yes	WFO	250	Aug 26	Yes
AIPL15449	Bethireddy Ravindra Kumar	Samsung	31 Aug 2026	Afternoon Shift (Starts anytime between 13:00 and 15:59 )	Yes	WFO	250	Aug 26	Yes


20. How do you secure a Terraform state file?