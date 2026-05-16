# 🚀 Section9 Devops — Interview Master Guide

> 🧠 Styled for fast revision + deep understanding

---

## 📌 Overview
# .NET Interview Preparation - Section 9: DevOps Basics

## 13 Years Experience Candidate Answers

---

## Question 1: Explain CI/CD pipeline. What are the stages in a typical CI/CD pipeline?

**Answer:**

**CI (Continuous Integration):**
- Developers commit code frequently
- Automated builds and tests on each commit
- Immediate feedback on quality

**CD (Continuous Deployment/Delivery):**
- Automates release process
- Deploy to production automatically after CI passes

**Typical stages:**
1. **Source** - Code commit triggers pipeline
2. **Build** - Compile, package
3. **Test** - Unit, integration, other tests
4. **Staging** - Deploy to test environment
5. **Production** - Deploy to live

**Quality gates:** Each stage must pass before proceeding. Canary deployments, rollback capabilities.

---

## Question 2: What is Docker? How does it work? Explain containers vs virtual machines.

**Answer:**

**Docker:**
- Platform for containerization
- Packages application with all dependencies
- Runs consistently across environments

**How it works:**
- Dockerfile defines image
- Image is template for container
- Container is running instance of image
- Shares OS kernel with host

**Containers vs VMs:**

| Aspect | Containers | VMs |
|--------|-----------|-----|
| OS | Share host kernel | Full OS |
| Size | MB | GB |
| Start time | Seconds | Minutes |
| Isolation | Process | Full |
| Overhead | Low | Higher |

**Containers are lighter, faster, more portable.**

---

## Question 3: How do you containerize a .NET application? What goes into a Dockerfile?

**Answer:**

**Dockerfile for .NET:**
```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY *.csproj ./
RUN dotnet restore
COPY . ./
RUN dotnet publish -c Release -o /src/publish

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /src
COPY --from=build /src/publish .
ENTRYPOINT ["dotnet", "myapp.dll"]
```

**Key points:**
- Multi-stage builds for smaller images
- Use specific version tags
- Don't run as root
- Use .dockerignore
- Expose ports (80, 443)

---

## Question 4: What is Docker Compose? How do you configure multi-container applications?

**Answer:**

**Docker Compose:**
- Tool for defining multi-container apps
- Uses docker-compose.yml
- Starts all services together

**Example:**
```yaml
version: '3.8'
services:
  web:
    build: .
    ports:
      - "8080:80"
    depends_on:
      - db
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
```

**Commands:**
- `docker-compose up` - start all
- `docker-compose down` - stop all
- `docker-compose build` - build images

---

## Question 5: Explain the difference between docker build, docker run, and docker-compose up.

**Answer:**

**docker build:**
- Creates Docker image from Dockerfile
- Syntax: `docker build -t myapp .`
- Only builds, doesn't run

**docker run:**
- Creates and starts container from image
- Syntax: `docker run -p 8080:80 myapp`
- One container at a time

**docker-compose up:**
- Builds (if needed) and starts entire app
- Reads docker-compose.yml
- Manages multiple containers together
- Orchestrates networking

**Use docker-compose** for multi-container apps, **docker run** for single container.

---

## Question 6: What are the best practices for .NET Docker images? Multi-stage builds?

**Answer:**

**Best practices:**
- Use official Microsoft base images
- Use specific versions, not 'latest'
- Use multi-stage builds (smaller images)
- .dockerignore to exclude files
- Don't run as root user
- Set WORKDIR appropriately

**Multi-stage builds:**
```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
RUN dotnet publish -c Release -o /out

# Runtime stage  
FROM mcr.microsoft.com/dotnet/aspnet:8.0
COPY --from=build /out .
```
- Final image only has runtime
- Smaller, more secure
- Build tools not in production image

---

## Question 7: Explain Kubernetes basics: Pod, Deployment, Service, Ingress.

**Answer:**

**Pod:**
- Smallest deployable unit
- One or more containers sharing storage/network
- Usually one container per pod

**Deployment:**
- Manages Pod replicas
- Handles updates, rollbacks
- Ensures desired number of pods running
```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 3
```

**Service:**
- Exposes pods internally
- Load balancing between pods
- Cluster-internal networking

**Ingress:**
- HTTP/HTTPS routing
- External access to services
- Host/path-based routing

---

## Question 8: What is the difference between Azure DevOps, GitHub Actions, and Jenkins?

**Answer:**

**Azure DevOps:**
- Microsoft's full DevOps platform
- Azure Pipelines for CI/CD
- Boards, Repos, Test Plans
- Integrated with Azure services

**GitHub Actions:**
- Built into GitHub
- Workflows in .github/workflows
- Great for GitHub-hosted repos
- Growing ecosystem

**Jenkins:**
- Open source, self-hosted
- Highly customizable
- Large plugin ecosystem
- More manual setup required

**Choose based on:** Existing tooling, team expertise, cloud provider.

---

## Question 9: How do you manage secrets in .NET applications? What are the options?

**Answer:**

**Options for secrets:**
- **Environment variables** - simple, not secure
- **Azure Key Vault** - production, Azure
- **AWS Secrets Manager** - production, AWS
- **HashiCorp Vault** - self-hosted
- **Kubernetes Secrets** - K8s deployments

**In .NET:**
```csharp
// Azure Key Vault
builder.Configuration.AddAzureKeyVault(
    new Uri("https://myvault.vault.azure.net/"),
    new DefaultAzureCredential());
```

**Best practices:**
- Never commit secrets
- Use secret management services
- Rotate secrets regularly
- Different secrets per environment

---

## Question 10: What is infrastructure as code? How does it apply to .NET deployments?

**Answer:**

**Infrastructure as Code (IaC):**
- Define infrastructure in code files
- Version control, review, automate

**Options for .NET:**
- **Terraform** - multi-cloud
- **ARM templates** - Azure
- **Pulumi** - code-based
- **Bicep** - simplified ARM

**Example (Terraform for Azure App Service):**
```hcl
resource "azurerm_app_service" "app" {
  name                = "myapp"
  resource_group_name = "myrg"
  app_service_plan_id = azurerm_app_service_plan.plan.id
}
```

**Benefits:** Reproducible, version-controlled, auditable, automated.

---

## Question 11: Explain the different Azure services for hosting .NET applications.

**Answer:**

**Azure App Service:**
- PaaS for web apps
- Managed scaling, deployment
- Easy to start with
- Good for most web apps

**Azure Container Apps:**
- Serverless containers
- Kubernetes-based, simpler
- Event-driven scaling
- Good for microservices

**Azure Kubernetes Service (AKS):**
- Managed Kubernetes
- Full control, more complexity
- Good for large microservices

**Azure VMs:**
- IaaS, full control
- Most work to manage
- For specific requirements

---

## Question 12: What is the difference between Azure App Service and Azure Container Apps?

**Answer:**

**Azure App Service:**
- Directly hosts .NET apps
- Platform manages runtime
- Simple deployment (ZIP, Git)
- Good for: web apps, APIs

**Azure Container Apps:**
- Runs containers
- Serverless billing (pay per use)
- Event-driven scaling
- Good for: microservices, event-driven

**Comparison:**

| Aspect | App Service | Container Apps |
|--------|-------------|----------------|
| Hosting | Managed runtime | Containers |
| Scaling | Manual/auto | Event-driven |
| Billing | Per instance | Per request |
| Complexity | Lower | Higher |
| Control | Less | More |

---

## Question 13: How do you implement logging and monitoring in production .NET applications?

**Answer:**

**Logging in .NET:**
- Use structured logging (Serilog, NLog)
- Log levels: Debug, Information, Warning, Error
- Include correlation IDs for tracing

```csharp
_logger.LogInformation("Processing order {OrderId}", orderId);
```

**Monitoring:**
- Send logs to centralized system
- **Application Insights** - Azure
- **Seq** - self-hosted
- **ELK Stack** - Elasticsearch, Logstash, Kibana
- **Datadog** - SaaS

**Key practices:**
- Don't log sensitive data
- Use meaningful log levels
- Include context
- Monitor metrics + logs + traces

---

## Question 14: What is Application Insights? How do you use it for performance monitoring?

**Answer:**

**Application Insights:**
- Azure's application performance monitoring
- Part of Azure Monitor

**Setup:**
```csharp
// Add to Program.cs
builder.Services.AddApplicationInsightsTelemetry();
```

**Features:**
- Request tracking
- Dependency tracking
- Exception tracking
- Performance monitoring
- Custom metrics

**Dashboard shows:**
- Request rates, response times
- Failed requests
- Dependencies performance
- Exception details

**Use for:** Troubleshooting, performance optimization, availability monitoring.

---

## Question 15: Explain the concept of blue-green deployment and canary releases.

**Answer:**

**Blue-Green Deployment:**
- Two identical environments
- Blue = current production
- Green = new version
- Switch traffic when green ready
- Instant rollback if issues
- Double resource requirement

**Canary Release:**
- Gradually shift traffic
- Start with 5%, then 25%, then 100%
- Test with small percentage first
- Rollback if issues
- Less resource overhead

**Azure:**
- App Service deployment slots for blue-green
- Azure Front Door for canary routing

---

## Question 16: How do you handle configuration across different environments?

**Answer:**

**Environment-specific config:**
- Development, Staging, Production
- Different connection strings, API keys

**Options:**
```csharp
// appsettings.json (base)
{
  "ConnectionStrings": { "Default": "..." }
}

// appsettings.Development.json
{
  "ConnectionStrings": { "Default": "dev-conn" }
}

// Use in code
builder.Configuration.GetConnectionString("Default");
```

**Other approaches:**
- Environment variables
- Azure App Configuration
- Key Vault for secrets

**Best practice:** Environment-specific files + override with environment variables.

---

## Question 17: What is Helm charts? How do you use it for Kubernetes deployments?

**Answer:**

**Helm:**
- Package manager for Kubernetes
- Charts = packages
- Templates for K8s resources

**Structure:**
```
mychart/
  Chart.yaml
  values.yaml
  templates/
    deployment.yaml
    service.yaml
```

**Commands:**
```bash
helm install myapp ./mychart
helm upgrade myapp ./mychart
helm rollback myapp 1
```

**Values.yaml** has configurable values, **templates/** have K8s manifests with templating.

**Benefits:** Versioning, rollback, reusability, templating.

---

## Question 18: Explain the difference between VMs, containers, and serverless for .NET.

**Answer:**

**Virtual Machines:**
- Full OS, complete control
- Most resource overhead
- Highest flexibility
- Most management work

**Containers:**
- Share OS kernel
- Lightweight, fast
- Portable
- Less flexible than VMs

**Serverless (Functions):**
- No server management
- Auto-scale to zero
- Pay per execution
- Limited to event triggers

**Use cases:**
- VMs: specific OS requirements, legacy
- Containers: microservices, DevOps
- Serverless: event-driven, cost-sensitive

---

## Question 19: How do you implement health checks for containerized .NET applications?

**Answer:**

**ASP.NET Core health checks:**
```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<DbContext>("database")
    .AddUrlGroup(new Uri("https://api.com"));
```

**Endpoint:**
```csharp
app.MapHealthChecks("/health");
```

**Kubernetes probes:**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 80
  initialDelaySeconds: 30
readinessProbe:
  httpGet:
    path: /ready
    port: 80
```

- **Liveness** - is container running?
- **Readiness** - can serve traffic?

---

## Question 20: What is the difference between vertical and horizontal scaling?

**Answer:**

**Vertical Scaling:**
- Add more resources to existing instance
- CPU, RAM, disk
- Same server, more power
- Limited by hardware
- Downtime when scaling

**Horizontal Scaling:**
- Add more instances
- Multiple smaller servers
- Load balancer distributes
- No single point of failure
- Scales infinitely (理论上)

**For .NET:**
- App Service: vertical scaling in tier, horizontal via instance count
- Container Apps: horizontal with replicas
- Kubernetes: horizontal pod autoscaling

**Common pattern:** Horizontal scaling is preferred for cloud-native.

---

*End of Section 9: DevOps Basics*

---

## 🎯 Key Takeaways
- Revise important concepts quickly
- Focus on interview-ready answers
- Practice explaining in your own words

---

## 🎤 Interview Tip
> Always explain **WHY + HOW**, not just definitions.
