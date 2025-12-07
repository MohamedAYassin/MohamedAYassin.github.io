---
title: "HoneyKube"
date: 2025-12-07T00:00:00Z
draft: false
author: "Mohamed Yassin"
tags:
  - Kubernetes
  - Security
  - AI
  - LLM
  - Honeypot
  - Deception
  - Python
  - Redis
  - Docker
image: /images/honey.png
description: "A deep technical analysis of HoneyKube - an AI-driven, Kubernetes-native honeypot system that leverages LLMs to create adaptive, high-interaction deception environments."
toc: true
github: https://github.com/MohamedAYassin/HoneyKube
---

HoneyKube represents a paradigm shift in honeypot architecture. 

Traditional honeypots rely on static templates or low-interaction emulation. HoneyKube leverages Large Language Models (LLMs) and Kubernetes native features to create a high-interaction, adaptive deception environment.

This post analyzes the technical implementation. We examine the four microservices. We dissect the Python code. We review the Kubernetes manifests.

## 1. System Architecture

HoneyKube runs as a distributed system. It is designed for horizontal scalability.

### Core Components

* **Port Listener**: The ingress point. It accepts connections and proxies detailed metadata.
* **Scanner Detector**: A high-performance classification engine. It identifies attack tools.
* **LLM Planner**: The brain. It generates dynamic, context-aware responses via OpenRouter.
* **Artifact Sink**: The forensics unit. It stores logs and uploaded payloads.
* **Redis**: The state engine. It maintains session continuity and caching.

## 2. Service Deep Dive: Port Listener

The `port-listener` service is the front line. It handles raw TCP/HTTP traffic. It does not serve static files. It exists to deceive.

### Metadata Extraction

The service extracts forensic quality metadata from every request. This happens in `extract_request_metadata` inside `services/port-listener/main.py`.

```python
async def extract_request_metadata(request: web.Request) -> RequestMetadata:
    """Extract metadata from incoming request."""
    # Get client IP (handle proxies)
    peername = request.transport.get_extra_info("peername")
    src_ip = request.headers.get("X-Forwarded-For", "").split(",")[0].strip()
    if not src_ip and peername:
        src_ip = peername[0]
    src_port = peername[1] if peername else 0
  
    # Read body
    body = None
    body_size = 0
    try:
        body_bytes = await request.read()
        body_size = len(body_bytes)
        if body_size > 0:
            try:
                body = body_bytes.decode("utf-8", errors="replace")
            except Exception:
                body = f"[binary data: {body_size} bytes]"
    except Exception as e:
        logger.warning(f"Failed to read request body: {e}")
  
    # Extract headers
    headers = dict(request.headers)
  
    return RequestMetadata(
        src_ip=src_ip or "unknown",
        src_port=src_port,
        dst_port=LISTEN_PORT,
        method=request.method,
        path=request.path,
        headers=headers,
        query_params=dict(request.query),
        body=body,
        body_size=body_size,
        timestamp=get_timestamp(),
        raw_request=raw_request
    )
```

**Technical Note**: We handle `X-Forwarded-For` explicitly. This allows HoneyKube to run behind standard Kubernetes Ingress controllers or properly configured LoadBalancers without losing source IP visibility.

### CVE-2025-55182 Detection (React2Shell)

One of the most advanced features is the specific detection logic for the critical Next.js React Server Components RCE. We do not just log it. We emulate vulnerability.

This logic resides in `detect_rsc_exploit`. It checks for the `Next-Action` header and specific payload markers like `__proto__` or `process.mainModule`.

```python
def detect_rsc_exploit(metadata: RequestMetadata) -> Optional[LLMResponse]:
    """
    Detect React Server Components (RSC) exploit attempts (CVE-2025-55182).
    Returns a fake "vulnerable" response to make scanners think the exploit worked.
    """
    # must be POST and have Next-Action
    if metadata.method != "POST":
        return None
  
    has_next_action = any(k.lower() == "next-action" for k in headers.keys())
    if not has_next_action:
        return None
  
    # Check for RSC exploit patterns
    rsc_patterns = [
        "__proto__",
        "constructor:constructor", 
        "NEXT_REDIRECT",
        "process.mainModule",
        "child_process",
        "execSync",
    ]
  
    is_rsc_exploit = any(pattern in body for pattern in rsc_patterns)
  
    if is_rsc_exploit:
        # Return fake successful RCE response
        # The scanner checks for X-Action-Redirect containing /login?a=11111
        # (11111 = 41 * 271, the math in the exploit payload)
        return LLMResponse(
            status_code=303,
            headers={
                "Content-Type": "text/plain",
                "X-Powered-By": "Next.js",
                "X-Action-Redirect": "/login?a=11111", # The magic number
                "Location": "/login?a=11111",
            },
            body="",
            delay_ms=100,
            notes="RSC Exploit: Fake RCE success response (CVE-2025-55182)"
        )
```

This code snippet proves the high-interaction capability. We return the exact status code (`303`) and header (`X-Action-Redirect`) that automated scanners look for. This generates a "confirmed vulnerable" result on the attacker's dashboard.

## 3. Service Deep Dive: Scanner Detector

The `scanner-detector` service filters noise. It separates automated vulnerability scanners from manual attacks.

### Signature Database

The database is a Python dictionary map. It links regex patterns to scanner families.

```python
SCANNER_SIGNATURES: Dict[str, Dict] = {
    "sqlmap": {
        "family": ScannerFamily.VULN_SCANNER,
        "user_agents": [r"sqlmap", r"sqlmap/[\d.]+"],
        "headers": {"X-Sqlmap": r".*"},
        "path_patterns": [
            r".*['\"].*(?:or|and|union|select).*",
            r".*(?:benchmark|sleep|waitfor)\s*\(",
        ],
        "body_patterns": [r"sqlmap", r"--dbs", r"--tables", r"--dump"],
    },
    "burp": {
        "family": ScannerFamily.WEB_SCANNER,
        "user_agents": [r"burp", r"Burp.*"],
        "headers": {"X-Burp-.*": r".*"},
        "body_patterns": [r"burp.*collaborator", r"oastify\.com"],
    },
    "react2shell": {
        "family": ScannerFamily.EXPLOIT_FRAMEWORK,
        "user_agents": [r"Assetnote", r"assetnote"],
        "headers": {
            "Next-Action": r".*",
            "X-Nextjs-Request-Id": r".*",
        },
        "body_patterns": [
            r"process\.mainModule\.require",
            r"child_process.*execSync",
            r"constructor:constructor",
        ],
    },
}
```

### Exploit Pattern Categories

Beyond tool names, we classify the *type* of attack using `EXPLOIT_CATEGORIES`.

**Command Injection**:

```python
r";\s*(?:cat|id|whoami|uname|pwd|ls|dir|echo|wget|curl|nc|bash|sh|python|perl|ruby|php)"
r"\|\s*(?:cat|id|whoami|uname|pwd|ls|dir|echo|wget|curl|nc|bash|sh)"
r"(?:system|exec|shell_exec|passthru|popen|proc_open)\s*\("
```

**SQL Injection**:

```python
r"(?:union\s+(?:all\s+)?select|select\s+.*\s+from|insert\s+into|update\s+.*\s+set|delete\s+from)"
r"(?:'\s*or\s*'|'\s*and\s*'|or\s+1\s*=\s*1|and\s+1\s*=\s*1)"
```

**LFI (Local File Inclusion)**:

```python
r"(?:php://filter|php://input|expect://|data://)"
r"(?:/var/log/|/var/www/|/tmp/)"
r"\.(?:log|conf|ini|htaccess|htpasswd)$"
```

The detection logic (`detect_exploit_patterns`) iterates through these compiled regexes against every incoming request path, body, and query string.

## 4. Service Deep Dive: LLM Planner

This is the core innovation. Instead of static files, we prompt an LLM to hallucinate a response.

### System Prompt Engineering

The prompt is strictly engineered for JSON output and safety.

```python
SYSTEM_PROMPT = """You are an AI assistant that generates realistic HTTP responses for a honeypot system.
Your goal is to simulate a vulnerable server to attract and study attackers.

IMPORTANT RULES:
1. NEVER execute any actual commands - only simulate responses
2. Generate responses that APPEAR vulnerable but contain no real exploits
3. Include realistic error messages, stack traces, and information leaks
4. Match the fingerprint profile provided (server type, version, OS)

OUTPUT SCHEMA (strict JSON):
{
    "status_code": <int 100-599>,
    "headers": {"Header-Name": "value", ...},
    "body": "<response body as string>",
    "delay_ms": <int 0-10000>,
    "notes": "<internal diagnostics>"
}"""
```

### The "No 404" Fallback Logic

LLMs can be slow or unavailable. We implemented a robust fallback mechanism in `generate_fallback` and `generate_dynamic_response` (in `services/llm-planner/main.py`).

**Dynamic File Extension Handling**:
We determine content type by file extension and generate plausible dummy content. This ensures we *never* return a 404, which keeps attackers engaged.

```python
    elif ext in ['.php', '.phtml']:
        return {
            "status_code": 200,
            "headers": {"Content-Type": "text/html", "X-Powered-By": "PHP/7.4.3"},
            "body": f"""...
<!-- DB_HOST=localhost DB_USER=admin DB_PASS=P@ssw0rd123 -->
..."""
        }
  
    elif ext in ['.key', '.pem', '.crt']:
        return {
            "status_code": 200,
            "headers": {"Content-Type": "application/x-pem-file"},
            "body": f"""-----BEGIN RSA PRIVATE KEY-----
MIIEow{path_hash[:60]}...
-----END RSA PRIVATE KEY-----"""
        }
```

Notice the `path_hash` usage. We seed the fake data with a hash of the requested path. This ensures that if an attacker requests `/config/db.key` twice, they get the exact same "random" key both times. Consistency is key to deception.

## 5. Service Deep Dive: Artifact Sink

This service captures the evidence. It also performs "Exploit Stage Detection" to differentiate between a scanner just looking around and an attacker actively trying to break in.

### Exploit Stage Regex

We define specific patterns that indicate post-exploitation activity, such as data exfiltration or persistence.

```python
EXPLOIT_STAGE_PATTERNS = [
    # Post-exploitation commands
    r"(?:cat|type)\s+(?:/etc/passwd|/etc/shadow|C:\\Windows\\System32\\config)",
    r"(?:wget|curl|certutil|powershell).*(?:http|https|ftp)://",
  
    # Persistence mechanisms
    r"cron(?:tab)?|at\s+\d|schtasks|reg\s+add",
    r"(?:useradd|adduser|net\s+user)",
  
    # Data exfiltration
    r"tar\s+.*-[czf]|zip\s+-r|7z\s+a",
    r"scp\s+|rsync\s+|ftp\s+",
]
```

### File Upload Handling

When an attacker uploads a webshell or malware, we save it with its SHA-256 hash as the filename.

```python
        # Compute hash
        sha256 = compute_sha256(file_data)
      
        # Store artifact with hash as filename
        # This prevents directory traversal attacks and overwrites
        artifact_path = Path(ARTIFACT_DIR) / f"{sha256}_{safe_filename}"
      
        async with aiofiles.open(str(artifact_path), mode="wb") as f:
            await f.write(file_data)
      
        # Mark session as staging (file upload is typically staging)
        await redis_manager.mark_staging(src_ip, dst_port)
```

## 6. Kubernetes Implementation

HoneyKube is cloud-native. It relies on standard K8s objects.

### ConfigMaps for Fingerprints

We inject service profiles via ConfigMaps. This allows changing the honeypot's "personality" without rebuilding images.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fingerprint-wordpress
data:
  fingerprint.yaml: |
    name: wordpress-blog
    port: 8000
    server_header: "Apache/2.4.41 (Ubuntu)"
    vuln_tags: [wordpress, php, mysql]
    version: "5.8"
```

### Horizontal Pod Autoscaling (HPA)

Traffic to honeypots can be bursty (mass scanning). We use HPA to scale the lightweight port-listeners.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: port-listener-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: port-listener-wordpress
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Resource Limits

Security is paramount. We restrict the resources available to the containers to prevent them from affecting the host node if compromised (though they contain no shell, better safe than sorry).

```yaml
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
```

## 7. Redis State Management

Code in `shared/redis_client.py` handles the state.

**Session Tracking**:
We use a Redis Hash to store session data. Key format: `session:{src_ip}:{dst_port}`.

* `request_count`: Incremented on every hit.
* `fingerprint_leaks`: A Set of info already revealed (e.g., "we already showed them we are Apache 2.4").
* `prior_paths`: A List of the last 10 paths visited.

**Scanner Cache**:
To save CPU, we cache scanner detection results. Key format: `scanner:{src_ip}`. TTL is set to 1 hour.

## 8. Deployment Automation

The `k8s/deploy.sh` script simplifies the complex deployment process. It intelligently detects the environment.

**Environment Detection**:

```bash
    # Detect Kubernetes environment
    K8S_ENV="unknown"
    if command -v minikube &> /dev/null && minikube status &> /dev/null; then
        K8S_ENV="minikube"
        eval $(minikube docker-env) # Direct docker build into minikube
    elif command -v kind &> /dev/null && kind get clusters &> /dev/null 2>&1; then
        K8S_ENV="kind"
        # Kind requires loading images manually
    fi
```

**Image Loading for Kind**:
If using `kind`, we must explicitly load the built images, or K8s will try to pull them from Docker Hub and fail.

```bash
    if [ "$K8S_ENV" == "kind" ]; then
        CLUSTER_NAME=$(kind get clusters | head -1)
        kind load docker-image honeykube/scanner-detector:latest --name "${CLUSTER_NAME}"
        # ... logic repeats for all 4 images
    fi
```

## 9. Conclusion

HoneyKube demonstrates that honeypots can be intelligent. By combining the scalability of Kubernetes with the generative capabilities of LLMs, we create a trap that is harder to identify and richer in data.

The system is open source. The architecture is modular. The deployment is automated.

This is the future of deception technology.
