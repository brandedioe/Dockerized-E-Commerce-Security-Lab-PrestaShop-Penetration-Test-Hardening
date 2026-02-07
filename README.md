
# 🛒 PrestaShop Security Ops: Deployment, Hardening & Pentesting

![Docker](https://img.shields.io/badge/Infrastructure-Docker-blue?logo=docker)
![Security](https://img.shields.io/badge/Ops-Red%20%26%20Blue%20Team-red)
![Status](https://img.shields.io/badge/Status-Completed-success)

## ⚡ Executive Summary
This project simulates a real-world DevSecOps scenario: deploying a vulnerable e-commerce stack, hardening it against common threats, and auditing its security posture through active exploitation.

I deployed a **LAMP stack** (Linux, Apache, MySQL, PHP) using Docker containers, resolved critical infrastructure conflicts regarding Linux permissions and networking, and performed a **credential-based penetration test** to validate the system's resilience.

---

## 🏗️ Architecture & Tools
* **Container Orchestration:** Docker & Docker Network (Custom Bridge).
* **Target:** PrestaShop 9.0.3 (Modern E-Commerce Platform).
* **Database:** MySQL 5.7 (Persistent Volume).
* **Attack Vector:** THC-Hydra (Credential Stuffing/Brute Force).
* **OS:** Kali Linux.

---

## 🔧 Phase 1: Infrastructure Deployment & Troubleshooting
*The deployment was not straightforward. Standard container deployment resulted in a crash loop, requiring manual intervention.*

### The Challenge: "Error 500" Loop
Upon linking the application container to the MySQL database, the application failed to initialize, throwing a persistent **HTTP 500 Internal Server Error**.

### The Diagnosis
I inspected the container logs and identified a **permission drift** between the host machine (Kali Linux) and the container's Apache user (`www-data`). The container could not write to its own `/var/www/html` volume to generate cache files.

### The Fix
I manually overrode the container's file ownership and forced a cache clear to restore stability:

```bash
# 1. Force recursive ownership update
sudo docker exec -u root some-prestashop chown -R www-data:www-data /var/www/html

# 2. Purge corrupted cache artifacts
sudo docker exec -u root some-prestashop rm -rf /var/www/html/var/cache

# 3. Restart container to initialize clean state
sudo docker restart some-prestashop

```

---

## 🛡️ Phase 2: Security Hardening (Blue Team)

Once the system was stable, I performed a security audit to lock down default vulnerabilities before they could be exploited.

1. **Attack Surface Reduction:**
* **Action:** Renamed the default `/admin` login path to `/admin_secure`.
* **Impact:** Prevents automated bots and scanners from easily locating the administrative portal.


2. **Installer Remediation:**
* **Action:** Removed the `/install` directory immediately after setup.
* **Impact:** Prevents attackers from re-triggering the installation script to overwrite the database.


3. **Vulnerability Patching:**
* **Action:** Audited the *Module Manager* and identified 3 outdated plugins. Applied security patches to bring them to the latest stable versions.



---

## ⚔️ Phase 3: The Attack Simulation (Red Team)

To validate the strength of the administrative credentials, I switched roles to perform a **Black Box Penetration Test**.

* **Objective:** Compromise the Administrator account.
* **Vector:** HTTP POST Brute Force.
* **Tool:** Hydra.

### Methodology

I identified the custom login endpoint (`/admin_secure/login`) and crafted a targeted wordlist. I then executed a `http-post-form` attack to bypass the login form.

**Attack Command:**

```bash
hydra -l israeloluwasegun57@gmail.com -P pass.txt -s 8080 -t 4 localhost http-post-form "/admin_secure/index.php:email=^USER^&passwd=^PASS^&submitLogin=1:F=Invalid password"

```

### Results

**Critical Failure Identified.** The system allowed unlimited login attempts without IP banning or progressive delays. Hydra successfully recovered the password (`D@mil@re!6`) in under 12 seconds.


---

## 💡 Key Takeaways & Mitigation

This project highlighted the critical gap between "deployment" and "secure deployment."

**Recommendations for Production:**

1. **WAF Implementation:** Deploy ModSecurity to block repeated login failures (Rate Limiting).
2. **MFA Enforcement:** Mandatory 2FA for all Back Office access.
3. **Least Privilege:** Ensure container volumes are mounted with strict user permissions from the start to avoid the 500 Error loop encountered during deployment.
