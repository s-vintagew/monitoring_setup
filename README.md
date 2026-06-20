# 🚀 Homelab Observability Stack & GitOps Pipeline

An infrastructure-as-code (IaC) repository deploying a comprehensive, automated observability stack on a Fedora-based homelab. This project transitions manual system administration into a fully automated GitOps workflow using GitHub Actions and self-hosted runners.

## 🏗️ Architecture Overview & Linkage

This project represents the **Observability and Operations Layer** of my broader homelab architecture. It is designed to directly monitor, scrape, and visualize the health of the media and download services established in my primary infrastructure repository.

🔗 **Upstream Infrastructure:** This stack monitors the services deployed in the [homelab-download-hub](https://github.com/s-vintagew/homelab-download-hub).

Rather than deploying monitoring tools blindly, this stack was built to integrate specifically with the `homelab-download-hub`'s Docker networks, pulling metrics from qBittorrent, Apache, and the host OS itself to create a "single pane of glass" for the entire server environment.

---

## 📊 Monitoring Setups Implemented

The observability stack relies on two primary data streams—**Metrics** and **Logs**—unified under a single visualization frontend.

* **Prometheus (Metrics Aggregation):** Configured to scrape time-series data from various endpoints, including the host system (Node Exporter), the web server (Apache Exporter), and download clients (qBittorrent).
* **Grafana (Visualization):** Serves as the central dashboarding tool, pulling data sources from both Prometheus and Loki to map out network traffic, download speeds, and server health.
* **Grafana Loki & Promtail (Log Aggregation):** Implemented to handle high-volume log data. Promtail actively scrapes local container and system logs (like Apache access logs) and ships them to Loki. 
  * *Design Choice:* By pushing Apache IP access logs to Loki rather than Prometheus, we successfully avoided the "high-cardinality" memory issues that often crash Prometheus instances.

---

## 🔧 Engineering Challenges & SRE Solutions

Building an integrated stack across different isolated Docker networks presented several networking and authentication challenges. Here is how they were resolved:

### 1. qBittorrent: Beating IP Bans & `Connection Refused` Errors
* **The Challenge:** When Prometheus attempted to scrape qBittorrent for metrics, qBittorrent's anti-brute-force security flagged the rapid, repeated requests. This resulted in the Prometheus container's IP being banned, throwing constant `connection refused` errors.
* **The Solution:** First, the WebUI network interface was updated to bind to `0.0.0.0` to ensure it was listening across the Docker bridge. Second, instead of relying on fragile plaintext credentials, I implemented **Zero-Touch Authentication** by configuring qBittorrent to "Bypass authentication for clients in permitted IP subnets." By whitelisting the internal Docker subnet, Prometheus can scrape metrics seamlessly while external traffic remains fully secured.

### 2. Apache: Firewall Drops and `mod_status` Permissions
* **The Challenge:** The Apache Exporter failed to gather metrics from the server because Apache's internal security blocked the scraper, and the Fedora host's firewall dropped the internal Docker routing.
* **The Solution:** Configured Apache's `Require ip` directives specifically to allow read-only access to `mod_status` from the Prometheus container's subnet. Additionally, updated Fedora’s `firewalld` rules to permit internal Docker network traversal without exposing the metrics endpoint to the public internet.

### 3. Grafana: Broken Dashboards and Missing Datasources
* **The Challenge:** After importing standard community dashboards for the exporters, the panels remained completely empty despite Prometheus gathering data. 
* **The Solution:** The JSON models of the imported dashboards contained hardcoded IDs and missing variable mappings. I repaired the variable queries (fixing the `label_values` regex to match my specific instance names) and manually created the missing `DS_LOCALHOST` datasource variable to correctly link the dashboard panels to my Prometheus backend.

---

## 🤖 Implementation of GitOps (Post-Infra Setup)

Once the infrastructure was stable, maintaining the stack via manual `docker compose` commands became inefficient. I migrated the entire deployment process to an automated GitOps pipeline.

### The "Pull Model" via Self-Hosted Runner
To maintain strict security on the Fedora host, I refused to open SSH ports (Port 22) to the internet or GitHub's cloud runners. Instead, I deployed a **Self-Hosted GitHub Actions Runner** locally. The runner securely polls GitHub via outbound HTTPS. When a commit is pushed, the server pulls the code down to itself, completely eliminating inbound firewall vulnerabilities.

### Pipeline Architecture (`deploy.yml`)
The automated CI/CD pipeline is built with strict pre-deployment checks to prevent broken configurations from taking down the monitoring stack:

1. **CI Validation Phase (`validate-configs`):**
   Before touching the live server, the pipeline uses the official `prom/prometheus` Docker image to run `promtool check config prometheus.yml`. This catches YAML syntax errors and invalid scrape jobs immediately.
2. **Workspace Synchronization:** 
   Utilizes `actions/checkout@v4` to pull the verified repository files into the runner's isolated workspace.
3. **Zero-Downtime Deployment:** 
   The pipeline securely copies the validated files into the production directory (`~/monitoring_setup_deploy`) and executes `docker compose up -d --remove-orphans`, smoothly restarting only the containers whose configurations have changed.