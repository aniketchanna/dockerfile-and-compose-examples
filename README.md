# dockerfile-and-compose-examples

Production Docker setup for a Java web application stack — custom Tomcat and Apache2 images with a Docker Compose orchestration file. This setup is used in a real production environment at BizmerlinHR (ClayHR) running on AWS Graviton (ARM64) instances.

---

## 📁 Repository structure

```
dockerfile-and-compose-examples/
├── Dockerfile.tomcat        # Custom Tomcat image with production JVM configuration
├── Dockerfile.apache2       # Custom Apache2 image configured as reverse proxy + SSL terminator
└── docker-compose.yml       # Orchestrates both services on a shared bridge network
```

---

## 🏗️ Architecture

```
Internet (HTTP/HTTPS)
        │
        ▼
  ┌─────────────┐
  │   Apache2   │  ports 80, 443  ← SSL termination + reverse proxy
  │  (512MB RAM)│
  └──────┬──────┘
         │ internal network (appnet)
         ▼
  ┌─────────────┐
  │    Tomcat   │  expose 8080, 8009  ← NOT public-facing
  │   (6GB RAM) │
  └─────────────┘
         │
    appnet (bridge)
```

Apache2 handles all public traffic and terminates SSL. It forwards requests internally to Tomcat over the `appnet` bridge network. Tomcat is never directly exposed to the internet.

---

## 🔍 Key details

### docker-compose.yml

**Tomcat service:**
- Image pulled from `.env` variable `${TOMCAT_IMAGE}` — allows environment-specific image versions without editing the compose file
- JVM tuned for production: G1GC, 1GB–4GB heap, heap dump on OOM, GC logging with rotation
- Conf, logs, and logback config mounted from host for easy updates without rebuilding the image
- Memory hard-limited to 6GB via `mem_limit`
- Built for ARM64 (AWS Graviton `t4g` instances) via `platform: linux/arm64`

**Apache2 service:**
- Depends on Tomcat — starts only after Tomcat is up
- SSL certificates mounted from host (`.crt` and `.key`) — no certs baked into image
- AJP workers config mounted for Tomcat integration
- Memory limited to 512MB
- Exposes ports 80 (HTTP) and 443 (HTTPS) to host

**Networking:**
- Both services on a shared `appnet` bridge network
- Tomcat only uses `expose` (internal only) — not accessible from outside the host
- Apache2 uses `ports` — accessible externally

---

## 🚀 How to run

**Step 1 — Create a `.env` file** in the same directory:

```bash
TOMCAT_IMAGE=your-ecr-repo/tomcat:latest
APACHE_IMAGE=your-ecr-repo/apache2:latest
```

**Step 2 — Create required host directories:**

```bash
mkdir -p /opt/clayhr-docker/tomcat/conf
mkdir -p /opt/clayhr-docker/tomcat/logs
mkdir -p /opt/clayhr-docker/tomcat/lib
mkdir -p /opt/clayhr-docker/apache2/certs
mkdir -p /opt/clayhr-docker/apache2/conf
```

**Step 3 — Place your SSL certs and config files:**

```bash
# SSL certs
/opt/clayhr-docker/apache2/certs/clayhr.crt
/opt/clayhr-docker/apache2/certs/clayhr-root.crt
/opt/clayhr-docker/apache2/certs/clayhr.key

# AJP workers config
/opt/clayhr-docker/apache2/conf/workers.properties

# Tomcat conf directory and logback config
/opt/clayhr-docker/tomcat/conf/
/opt/clayhr-docker/tomcat/lib/logback.xml
```

**Step 4 — Start the stack:**

```bash
docker compose up -d
```

**Step 5 — Verify both containers are running:**

```bash
docker compose ps
docker compose logs -f tomcat
docker compose logs -f apache2
```

---

## ⚙️ JVM tuning explained

The Tomcat `CATALINA_OPTS` are configured for a production Java workload:

| Option | Purpose |
|---|---|
| `-Xms1024m -Xmx4096m` | Initial and maximum heap size |
| `-XX:+UseG1GC` | G1 garbage collector — better for large heaps |
| `-XX:+ExitOnOutOfMemoryError` | Fail fast on OOM rather than running degraded |
| `-XX:+HeapDumpOnOutOfMemoryError` | Capture heap dump for post-mortem analysis |
| `-XX:HeapDumpPath=...` | Save heap dump to mounted log volume |
| `-Xlog:gc*:file=...` | GC log with 10 rotating files, 20MB each |

To add a metaspace cap (uncomment in compose file):
```
-XX:MaxMetaspaceSize=768m
```

---

## 🖥️ Infrastructure context

This stack runs on AWS EC2 Graviton (`t4g`) instances — ARM64 architecture. The `platform: linux/arm64` directive in the compose file ensures Docker pulls/runs the correct image architecture.

Both images are built and stored in AWS ECR. Image versions are managed via the `.env` file, allowing zero-edit deployments across environments (staging, production).

---

## 📌 Related repos

- [aws-cloudformation-templates](https://github.com/aniketchanna/aws-cloudformation-templates) — CloudFormation templates for the underlying AWS infrastructure
- [linux-automation-scripts](https://github.com/aniketchanna/linux-automation-scripts) — Server automation scripts used alongside this stack
- [cicd-pipeline-examples](https://github.com/aniketchanna/cicd-pipeline-examples) — CI/CD pipelines that build and deploy these images

---

## 👤 Author

**Aniket Channa** — Senior DevOps Engineer  
8 years experience · AWS (70%) · Azure/GCP (30%) · Open to remote worldwide  
[LinkedIn](https://linkedin.com/in/aniketchanna) · [GitHub](https://github.com/aniketchanna) · IST (UTC+5:30)
