# node-app — Intentionally Vulnerable Node.js Security Lab

A deliberately vulnerable Node.js application built for DevSecOps training,
security scanning demonstrations, and CI/CD pipeline security gate validation.
Designed to showcase OWASP Top 10 vulnerabilities and demonstrate how tools
like Trivy, npm audit, SonarQube, and Jenkins catch them in a real pipeline.

---

## Purpose

This repo serves two goals:

1. **Security training** — demonstrates real OWASP Top 10 vulnerabilities
in a safe, controlled environment with clear annotations on every flaw
2. **DevSecOps pipeline demo** — shows how a Jenkins CI/CD pipeline with
Trivy image scanning, npm audit, and SonarQube integration catches
vulnerabilities at each stage before they reach production

---

## Repository Structure

```
node-app/
├── app.js                    # Express API with intentional vulnerabilities
├── Dockerfile                # Vulnerable container — intentional for lab use
├── Jenkinsfile               # Multi-stage CI/CD pipeline with security gates
├── package.json              # Dependencies — intentionally outdated versions
└── tests/
    └── sample.test.js        # Jest test suite
```

---

## Application — app.js

Express.js REST API running on port 3000 with the following endpoints:

| Endpoint | Method | Purpose | Vulnerability |
|---|---|---|---|
| / | GET | Home — returns app info | Exposes API key in response |
| /health | GET | Health check | Clean — safe endpoint |
| /user | GET | Fetch user by ID | SQL Injection via query param |
| /login | POST | User authentication | SQL Injection, JWT secret exposed |
| /search | GET | Search users | SQL Injection via search term |
| /admin/users | GET | List all users | No authentication — open to anyone |
| /ping | POST | Ping a host | Command Injection via host param |
| /user/:id | DELETE | Delete user | IDOR — no authorization check |

---

## Documented Vulnerabilities

### Application Layer (app.js)

**V1 — Hardcoded Credentials**
Database host, username, password, API key, and JWT secret all hardcoded
directly in source code — visible to anyone with repo access.

**V2 — Secret Logging**
API key and JWT secret printed to stdout on server startup and on every
database connection — appears in all log aggregation systems.

**V3 — Sensitive Data Exposure**
GET / returns API key and database connection details in plain JSON response
to any unauthenticated caller.

**V4, V5, V6 — SQL Injection (three endpoints)**
/user, /login, and /search all concatenate raw user input directly into
SQL queries. Classic injection: ' OR '1'='1 bypasses /login entirely.

**V7 — Missing Authentication**
/admin/users requires no token, session, or credential — returns full user
table to any caller with network access.

**V8 — Command Injection**
/ping passes host parameter directly to exec() with no sanitization.
Payload: 8.8.8.8; cat /etc/passwd executes arbitrary OS commands.

**V9 — Insecure Direct Object Reference (IDOR)**
DELETE /user/:id performs no authorization check — any caller can delete
any user record by guessing or enumerating IDs.

---

### Container Layer (Dockerfile)

| # | Vulnerability | Detail |
|---|---|---|
| 1 | EOL base image | node:14 — end of life, no security patches |
| 2 | Hardcoded secrets in ENV | DB_PASSWORD, API_KEY, JWT_SECRET baked into image |
| 3 | Secrets in ARG | MYSQL_ROOT_PASSWORD visible in docker history |
| 4 | Unnecessary packages | vim, telnet, netcat, wget increase attack surface |
| 5 | Outdated dependencies | express 4.17.1, mysql 2.18.1 — known CVEs |
| 6 | Dev deps in production | nodemon, jest installed in production image |
| 7 | Unrestricted COPY | All files including .env copied into image |
| 8 | Credentials written to files | credentials.txt and .env written inside image layer |
| 9 | Unnecessary ports exposed | Ports 22 and 3306 exposed alongside app port 3000 |
| 10 | Running as root | No USER instruction — container runs as root |
| 11 | No HEALTHCHECK | Docker has no native health awareness |
| 12 | No resource limits | No memory or CPU constraints defined |

---

### Pipeline Layer (Jenkinsfile)

| Vulnerability | Location |
|---|---|
| Hardcoded DOCKER_PASSWORD in environment block | environment {} |
| Hardcoded API_KEY in environment block | environment {} |
| Plain text docker login using password variable | Push to Registry stage |
| Secret printed to console log on deploy | Deploy stage echo |
| Quality gate disabled — allows vulnerable code through | Quality Gate stage |
| npm audit runs with || true — failures ignored | Security Scan stage |
| Trivy scan runs with || true — failures ignored | Security Scan Docker stage |

---

## Jenkins Pipeline Stages

```
Checkout
    └── Pull source from SCM

Install Dependencies
    └── npm install

Run Tests
    └── npm test --coverage (Jest)

Quality Gate
    └── SonarQube analysis (commented out — intentionally disabled for lab)

Security Scan — Dependencies
    └── npm audit --json > npm-audit.json

Build Docker Image
    └── docker build -t node-app:${BUILD_NUMBER} .
    └── docker tag node-app:${BUILD_NUMBER} node-app:latest

Security Scan — Docker Image
    └── Trivy scan: HIGH and CRITICAL severity on built image

Test Docker Container
    └── docker run -d -p 3000:3000 node-app:${BUILD_NUMBER}
    └── curl -f http://localhost:3000/health
    └── docker stop + docker rm

Push to Registry (main branch only)
    └── docker login registry.company.com
    └── docker push node-app:${BUILD_NUMBER}

Deploy (main branch only)
    └── docker stop node-app-prod
    └── docker run -d -p 80:3000 --restart unless-stopped node-app:${BUILD_NUMBER}

Post Actions
    └── always: docker system prune -f
    └── success: build summary
    └── failure: failure notification
```

---

## Tech Stack

| Component | Version | Status |
|---|---|---|
| Node.js base image | 14 | EOL — intentionally outdated |
| Express | 4.17.1 | Known CVEs — intentional |
| MySQL client | 2.18.1 | Known CVEs — intentional |
| body-parser | 1.19.0 | Outdated — intentional |
| lodash | 4.17.11 | Prototype pollution CVE — intentional |
| moment | 2.24.0 | Outdated — intentional |
| axios | 0.18.0 | SSRF CVE — intentional |
| jsonwebtoken | 8.5.1 | Known CVEs — intentional |
| Jest | 30.2.0 | Test runner |
| nodemon | 1.19.1 | Dev only |

---

## Running Locally

```bash
# Install dependencies
npm install

# Start the app
npm start
# Server running on http://localhost:3000

# Development mode with auto-reload
npm run dev

# Run tests with coverage
npm test
```

---

## Running with Docker

```bash
# Build image
docker build -t node-app:latest .

# Run container
docker run -d --name node-app -p 3000:3000 node-app:latest

# Verify health
curl http://localhost:3000/health

# View logs
docker logs node-app

# Stop and remove
docker stop node-app && docker rm node-app
```

---

## Security Scanning

```bash
# Scan for dependency vulnerabilities
npm audit

# Full audit report in JSON
npm audit --json > npm-audit.json

# Scan Docker image with Trivy
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image --severity HIGH,CRITICAL node-app:latest

# View docker history to see exposed build args
docker history node-app:latest
```

---

## Vulnerability Exploitation Examples (Lab Use Only)

```bash
# SQL Injection — bypass login authentication
curl -X POST http://localhost:3000/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin or 1=1 --","password":"anything"}'

# SQL Injection — dump all users via search
curl "http://localhost:3000/search?q=test"

# Command Injection — read system file
curl -X POST http://localhost:3000/ping \
  -H "Content-Type: application/json" \
  -d '{"host":"8.8.8.8"}'

# IDOR — delete any user without authentication
curl -X DELETE http://localhost:3000/user/1

# Unauthenticated admin access
curl http://localhost:3000/admin/users

# API key and DB credentials exposed in home route
curl http://localhost:3000/
```

---

## What a Secure Version Would Look Like

| Current Vulnerable Pattern | Secure Replacement |
|---|---|
| Hardcoded secrets in source | Vault or AWS Secrets Manager at runtime |
| node:14 EOL base image | node:20-alpine — current LTS minimal image |
| SQL string concatenation | Parameterized queries via mysql2 prepared statements |
| No authentication on admin routes | JWT middleware with role-based access control |
| exec() with raw unsanitized input | Input validation plus allowlist, no shell execution |
| Root container user | USER node non-root instruction in Dockerfile |
| Dev dependencies in production image | Multi-stage Dockerfile builder vs runtime stage |
| Trivy and audit with || true | Exit code enforcement — pipeline fails on HIGH/CRITICAL |
| SonarQube quality gate disabled | Gate blocks merge on security hotspots and bugs |
| Secrets logged to stdout | Structured logging with secret scrubbing middleware |

---

## Related DevSecOps Tools Demonstrated

| Tool | Stage | Purpose |
|---|---|---|
| npm audit | CI — dependencies | Known CVE detection in packages |
| Trivy | CI — container | Image layer vulnerability scanning |
| SonarQube | CI — code (commented out) | SAST static application security testing |
| Jest | CI — tests | Unit testing and coverage reporting |
| docker history | Manual inspection | Expose secrets baked into build args |
| Jenkins | CI/CD orchestration | Multi-stage pipeline with security gates |

---

## Learning Objectives

After working through this lab you will be able to:

- Identify OWASP Top 10 vulnerabilities in a real Node.js codebase
- Run Trivy image scans and interpret HIGH/CRITICAL findings
- Run npm audit and understand CVE severity scoring
- Explain why hardcoded secrets in Dockerfiles appear in docker history
- Describe how parameterized queries prevent SQL injection
- Configure Jenkins pipeline stages to enforce security gates
- Articulate the difference between SAST, DAST, and SCA scanning

---

## Author

Vikas Dhamija — Senior DevOps Engineer
GitHub: https://github.com/Vikas-DevOps-Git

---

> WARNING: This application is intentionally insecure.
> Do NOT deploy to any public-facing or production environment.
> For security training and CI/CD pipeline demonstration purposes only.
