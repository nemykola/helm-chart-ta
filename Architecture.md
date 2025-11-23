Landing Builder – AWS Architecture Proposal

This document provides a high-level overview of the proposed AWS architecture for the Landing Builder platform. It covers the system design, reasoning, service trade-offs, and future growth considerations.

 
1. Architecture Overview

The Landing Builder platform enables internal teams to create, preview, and publish marketing landing pages. The goal of this design is to deliver a scalable, highly available, secure, and low-maintenance infrastructure using fully managed AWS services.

High-Level Architecture

Compute
- ECS Fargate
Runs three independent container workloads:
    - editor (UI service)
	-	renderer (server-side rendering)
	-	publisher (background publishing worker)
Fargate eliminates the need to manage EC2 nodes or Kubernetes clusters, reducing operational overhead and maintenance time.

Storage
- Amazon S3 – stores uploaded assets, static content, and final published landing pages
-	Amazon CloudFront – global CDN accelerating content delivery and reducing ALB/ECS load
-	Amazon RDS  – stores drafts, templates, user data, and publishing metadata
-	Amazon ElastiCache  – optional caching layer for faster renderer responses

Networking
-	Application Load Balancer – handles HTTP routing to editor and renderer services
-	Amazon VPC – isolated, secure network across multiple Availability Zones

Supporting Services
-	AWS Secrets Manager – secure storage for credentials and API keys
-	Amazon SQS – asynchronous publishing queue
-	AWS IAM – least-privileged roles and permissions
-	Amazon CloudWatch – logging, metrics, dashboards, alarms
  

2. Reasoning and Assumptions

Assumptions
-	Editor and renderer may experience traffic spikes during marketing campaigns.
-	Publisher jobs may run in bursts but remain mostly idle.
-	The team prefers minimal DevOps overhead, so managed services are ideal.
-	Drafts and metadata require relational operations so SQL database recommended.
-	Published content benefits from global CDN caching.

 

Major Design Reasoning

Why ECS Fargate?
 - 	Zero cluster/node management
 - 	Predictable pay-per-use pricing
 - 	Auto-scaling based on CPU/memory
 - 	Natural service isolation (each task definition isolated)
 - 	Lower operational complexity vs. EKS or EC2

Why RDS PostgreSQL?
 - 	Strong relational model for metadata
 - 	Built-in HA with Multi-AZ
 - 	Automated backups, failover, and snapshot recovery
 - 	Easier onboarding for developers

Why S3 + CloudFront?
 - 	S3 provides 99.999999999% durability
 - 	CloudFront accelerates delivery and offloads traffic
 - 	Dramatically reduces renderer load
 - 	Ideal for static published content

Why SQS for Publishing Workflow?
 - 	Decouples the publishing pipeline from rendering
 - 	Smooths burst traffic
 - 	Publisher workers can auto-scale based on queue depth
 - 	Improves resilience and fault isolation

Security Decisions
 - 	IAM roles follow the principle of least privilege
 - 	All secrets stored in Secrets Manager, not in code
 - 	Only ALB is public; all ECS tasks run in private subnets
 - 	S3 bucket locked using Origin Access Control (OAC)
 - 	TLS termination via ACM certificates (CloudFront + ALB)

Reliability & Scalability
 - 	Multi-AZ across all core components
 - 	ECS auto-scaling for editor/renderer tasks
 - 	SQS queue-based scaling for publisher workers
 - 	ALB health checks ensure zero-downtime rolling updates
 - 	CloudFront + S3 absorb most global read traffic

 

3. AWS Services Used and Trade-offs

### Compute Options

| Service         | Pros                                           | Cons                                            | Decision     |
|-----------------|------------------------------------------------|--------------------------------------------------|--------------|
| ECS Fargate     | No servers, low maintenance, easy autoscaling | Higher cost under sustained heavy load          | Chosen       |
| EKS             | Highly flexible and portable                  | Requires cluster upgrades, node mgmt, SRE staff | Not needed   |
| EC2 Auto Scaling| Lowest cost at scale                          | Full AMI/patching/scaling burden                | Rejected     |



### Database Options

| Service            | Pros                                        | Cons                                 | Decision           |
|--------------------|---------------------------------------------|---------------------------------------|--------------------|
| RDS PostgreSQL     | Multi-AZ, strong relational support, backups| More expensive than self-managed       | Chosen             |
| DynamoDB           | Very low latency, massive scale             | Weak fit for relational metadata      | Future option      |
| Aurora Serverless  | Auto-scaling SQL, fast failover             | Complexity + higher cost              | Later growth path  |



Storage & Delivery Trade-offs
S3 + CloudFront chosen over serving static files from renderer because:
 - 	Cheaper
 - 	Faster global performance
 - 	Removes load from ECS
 - 	Simplifies renderer logic

 

4. Expanded Monitoring and Observability

A strong monitoring strategy is critical for operational visibility and reliability.

Logging
 - 	CloudWatch Logs gather stdout/stderr from all ECS containers
 - 	Log groups separated per service:
 - 	/landing/editor
 - 	/landing/renderer
 - 	/landing/publisher
 - 	Optional log forwarding to:
 - 	Amazon OpenSearch
 - 	Third-party  Splunk or Datadog

Metrics
 - 	ECS service metrics: CPU, memory, task count
 - 	ALB metrics: target response time, 4xx/5xx errors, request throughput
 - 	RDS metrics: CPU usage, read/write IOPS, free storage space
 - 	SQS metrics: queue depth, message age, DLQ count
 - 	CloudFront metrics: cache hit ratio, geographic distribution, error rates

Dashboards
 - 	CloudWatch dashboards showing:
 - 	ECS performance
 - 	RDS health
 - 	ALB traffic patterns
 - 	SQS queue depth trends
 - 	CloudFront real-time delivery metrics

Alerts
 - 	High CPU or memory on ECS services
 - 	ALB 5xx spikes
 - 	RDS low storage or high connections
 - 	SQS queue backing up
 - 	CloudFront cache hit ratio dropping (indicates misconfig or cold start)

Tracing (Optional)
 - 	AWS X-Ray for end-to-end tracing
 - 	Helps identify slow requests and bottlenecks
 - 	Particularly useful when renderer calls external services or Redis

This monitoring strategy ensures fast issue detection, system reliability, and actionable insights.

 

5. Expanded Migration and Future Growth

This architecture is intentionally designed with future-proofing in mind.

Database Evolution
 - 	Aurora Serverless  for on-demand, auto-scaling SQL
 - 	Read replicas for high-traffic read-heavy workloads
 - 	Option to move certain metadata to DynamoDB for ultra-low latency

Compute Evolution
 - 	Renderer and publisher could be rewritten as serverless Lambda functions
 - 	Lower idle cost
 - 	Instant scaling
 - 	Simplified deployment
 - 	Potential shift to EKS if cross-cloud portability or deeper custom orchestration is needed.

Global Delivery
 - 	Multi-region using Route53 latency routing
 - 	CloudFront already provides global reach; multi-region compute improves failover

Security Enhancements
 - 	Add AWS WAF for edge-level threat mitigation
 - 	Incorporate AWS Shield for DDoS protection
 - 	Use KMS customer-managed keys for S3 + RDS encryption

Observability Improvements
 - 	Move logs to OpenSearch for better search/analytics
 - 	Use Datadog / Prometheus if deeper container insights are needed

CI/CD Enhancements
 - 	GitHub Actions or AWS CodePipeline for:
 - 	Automated image builds
 - 	Image scanning
 - 	Blue/green or canary ECS deployments

This growth plan allows Landing Builder to evolve based on traffic, cost goals, and organizational needs.

 

Final Summary

This AWS architecture provides a strong foundation for the Landing Builder platform by offering:
 - 	High Availability through Multi-AZ Fargate and ALB
 - 	Scalability via autoscaling on ECS and SQS-driven workers
 - 	Security through IAM least privilege, Secrets Manager, and private networking
 - 	Operational Simplicity thanks to fully managed AWS services
 - 	Fast Global Delivery via CloudFront + S3
