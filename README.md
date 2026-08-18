# Architecture Exercise

## Architecture Diagram

![Architecture Diagram](dtex%20Architecture%20Diagram.png)

## Overview:

1. Using Microsoft Azure cloud platform, Azure services wherever possibile.
2. Assumption is that supplied JAR, .tar and Python application are working artefacts - focus on deployment / hosting as opposed to designing applications.

## CI/CD Pipelines

1. Use separate Azure DevOps pipelines for the Auth Service, Main App and Archive Job. This allows each component to be changed and deployed independently
2. Uniform architecture for easier maintenance
3. General deployment pattern: Azure DevOps -> build/test/security scans -> container image -> Azure Container Registry -> target Container App
4. Versioned images.
5. ACR not publicly exposed - private endpoint.
6. Use Azure DevOps self hosted virtual machine agent.
7. For security, run dependency/container vulnerability scanning before deployment.

## JAR - Highly available deployment authentication/authorization service (ref: Auth Service)

### Assumptions

1. Authentication service is supplied as a Java JAR from the existing CI pipeline
2. All requests from mobile application pass through this before reaching the main application
3. The service uses PostgreSQL for authentication related data
4. The service needs to be highly available
5. Public entry point into application.

### Architecture

1. The JAR will be packaged into a container image and deployed to Azure Container Apps.
2. Deployment:

Azure DevOps -> build/test JAR -> build container image -> Azure Container Registry -> Azure Container Apps

3. Azure Container Apps as it provides managed container hosting, ingress, autoscaling, health management and revisions without requiring the additional management overhead of Azure Kubernetes Service or virtual machines (more complexity and management).
4. The Auth Service will use external HTTPS ingress. Container Apps managed ingress/load balancing distribute requests across available replicas.
5. For the database, use Azure Database for PostgreSQL rather than hosting PostgreSQL by ourselves (easier to maintain). Public database access should be disabled and the Auth Service should connect privately - virtual networking / private endpoints throughout. 

### Scalability

1. Run multiple Auth Service replicas - avoid single instance
2. Container Apps can horizontally autoscale as traffic increases.
3. Managed ingress/load balancing distributes traffic across replicas

### Availability

1. Minimum two replicas to avoid relying on single application instance.
2. Health probes to identify unhealthy replicas.
3. Use a zone redundant Container Apps environment
4. PostgreSQL HA, zone redundant where supported, otherwise database remains a single point of failure.

### Maintainability

1. Managed Container Apps avoids managing VMs or Kubernetes infrastructure.
2. Container images are versioned in ACR - can roll back whenever required
3. Separate CI/CD pipeline allows the Auth Service to be deployed independently.

### Security

1. Managed services,  reduce the patching and infrastructure maintenance required from the team.
2. Only Auth Service has external HTTPS ingress.
3. PostgreSQL should not be publicly accessible; the Auth Service connects privately - virtual networking / private endpoints
4. Use TLS for database connections.
5. Prefer managed identity where supported and apply least privilege access/RBAC.
6. Credentials/secrets which are required should be held in Azure Key Vault rather than hardcoded.

### Cost Effectiveness

1. Container Apps avoids overhead of maintaining dedicated VMs or an AKS cluster where the additional control isn’t required.
2. Autoscaling allows compute to scale with actual demand
3. Managed PostgreSQL reduces the operational overhead of maintaining PostgreSQL ourselves.

## Highly available deployment of main application (ref: Main App)

### Assumptions

1. Node.js application supplied from CI as a .tar artefact.
2. Receives authenticated requests/connections through Auth Service
3. Reads from and writes to Cassandra.
4. Needs to be highly available.
5. Does not need to be directly internet accessible.

### Architecture

1. Use the same container deployment pattern as the Auth Service:
2. Azure DevOps -> .tar artefact -> build container image -> ACR -> Azure Container Apps
3. Because we already chose Azure Container Apps for auth service, use the same for the Node.js app. Gives us a consistent platform and keeps the architecture simpler.
4. Separate pipelines per service, but with the same reusable CI/CD architecture/pattern for consistency and maintainability.
5. The main app will have its own pipeline so it can be built and deployed independently
6. The application will use internal ingress only.  Auth Service can communicate with it, but isn’t directly exposed to mobile clients.
7. For Cassandra, use Azure Managed Instance for Apache Cassandra.
8. Cosmos DB for Apache Cassandra was considered, but as the brief specifically asks for Cassandra, Managed Instance was chosen as it runs native Cassandra and therefore reduces the risk of compatibility issues.
9. Keep the Cassandra backend private - private endpoint, virtual network

### Scalability

1. Multiple Main App replicas.
2. Horizontally autoscale as demand increases.
3. Cassandra can scale horizontally across multiple nodes.

### Availability

1. Multiple Main App replicas with health checks.
2. Managed load balancing across replicas.
3. Use availability zones where supported so loss of a node/zone does not take down the datastore

### Maintainability

1. Same Container Apps model as the Auth Service keeps the platform consistent
2. Both applications remain independently deployable.
3. Common deployment pattern.
4. Managed Cassandra reduces patching, scaling and node management compared with self hosting. Microsoft manages much of the deployment, scaling,  and maintenance.

### Security

1. Main App uses internal ingress and isn’t directly internet accessible.
2. Cassandra remains private and is only reachable by workloads that require it.
3. Main App connects to Cassandra privately - virtual networking / private endpoints.
4. Use managed identity where supported and Key Vault where credentials/secrets are required.
5. Apply least privilege access.

### Cost Effectiveness

1. Reusing Container Apps avoids introducing another hosting platform to operate
2. Autoscaling allows compute to increase as demand increases rather than overprovisioning 
3. Managed Cassandra has service cost, but reduces the engineering and overhead of managing the cluster ourselves.

## Task that runs periodically to archive old data (ref: Cleanup Job)

### Assumptions

1. Python task runs periodically rather than continuously
2. Its purpose is to archive old data from Cassandra, to offload storage cost
3. Archived data should be moved to long term storage. Colder.
4. Brief specifies archive, not delete, so I have not assumed that Cassandra records should automatically be deleted.

### Architecture

1. Run the Python application as a scheduled Azure Container Apps Job
2. No requirement for always running service, so scheduled compute
3. Schedule -> Container Apps Job -> read old data from Cassandra -> Azure Blob Storage
4. Azure Blob Storage is used as the cold archive destination as it provides durable, lower cost storage. Lifecycle management can also move older archived data into cheaper storage tiers via policies
5. Deployment: Azure DevOps -> build container image -> ACR -> update Container Apps Job
6. CI/CD runs when the Python application changes; the Container Apps Job schedule handles normal executions.

### Scalability

1. Avoid concurrent archive processes 
2. Can batch or parallelise runs if archive volumes increase.

### Availability

1. Configure retry
2. Monitor successful/failed executions
3. If deletion from Cassandra is later required, only delete after successful archival.

### Maintainability

1. Container Apps Job avoids maintaining an always running service.
2. Independent CI/CD pipeline.
3. Alert on repeated failures.

### Security

1. Job connects privately to Cassandra.
2. Blob Storage should not be publicly accessible - private endpoint
3. Prefer managed identity for Blob Storage access.
4. Give the job only the Cassandra/Storage permissions it requires.
5. Use Key Vault for any credentials which still need to exist.

### Cost Effectiveness

1. Compute is only required while the scheduled job is running rather than running a permanent service.
2. Moving historical data out of Cassandra and into blob storage provides cheaper long term storage.
3. Blob lifecycle policies can move older archived data into cheaper storage tiers over time
