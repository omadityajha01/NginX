# Prerequisites: what to learn to run and understand this project

This project turns 5 inputs (domain, ports, backends, backend port, protocol) into a working
NGINX gateway that runs in Docker. To work with it you need some **networking**, **NGINX**,
**Docker**, **TLS/HTTPS** and **Python** basics.

The topics are in two levels:

- **Level 1: just run it.** The minimum to get it working and test it.
- **Level 2: understand it.** What you need to read and change the templates and `setup.py`.

Each topic says where it appears in the project.

---

## Level 1: just run it (about 1–2 days)

### 1. Command line basics
- Opening a terminal (PowerShell / Git Bash) and moving between folders (`cd`, `ls`/`dir`)
- Running a program with arguments: `python setup.py --from-file`
- Reading error messages
- Editing the hosts file (`C:\Windows\System32\drivers\etc\hosts`) to test a custom domain locally

### 2. Docker essentials
- What an **image** and a **container** are, and how they differ
- Installing and starting **Docker Desktop** on Windows
- **Port mapping** (`"8080:80"` = port 8080 on your PC → port 80 in the container)
- **Docker Compose**: one `docker-compose.yml` that describes several containers
- Commands used here:
  - `docker compose up -d` / `docker compose down`
  - `docker compose logs -f nginx`
  - `docker compose exec nginx nginx -t` (test config) and `nginx -s reload`
  - `docker compose stop app2` (simulate a backend failure)

*Used in:* `docker-compose.yml`, `setup.py` (`start_stack`, `docker`)

### 3. HTTP basics
- What a request and a response are: method, URL, headers, body
- Status codes used here: `200`, `301` (redirect), `429` (too many requests), `502`/`503` (backend errors)
- Testing with **curl**: `curl -I`, and `curl -k` (accept a self-signed certificate)

### 4. Basic networking
- IP address, **port**, **hostname**, **domain**
- `localhost` / `127.0.0.1`, and why it means *the container itself* inside a container
  (which is why the script changes it to `host.docker.internal`)
- "Port already in use", and why the script suggests 8080 / 8443 instead

---

## Level 2: understand it (about 1–2 weeks)

### 5. NGINX fundamentals
- What a web server, a **reverse proxy** and a **load balancer** are
- Config structure: **directives**, **blocks/contexts** (`events`, `http`, `server`, `location`, `upstream`)
- `include` and the `nginx.conf` + `conf.d/*.conf` layout
- `server_name`, `listen`, and how NGINX picks a `server` block
- `location` matching (`location = /healthz` exact match vs `location /` prefix match)
- NGINX variables (`$host`, `$request_uri`, `$remote_addr`, `$upstream_addr`, …)
- The `map` directive (used for WebSockets and for skipping the cache)
- Workers: `worker_processes`, `worker_connections`
- Testing and reloading config with no downtime (`nginx -t`, `nginx -s reload`)

*Used in:* `templates/nginx.conf.tmpl`, `templates/site.conf.tmpl`

### 6. Reverse proxying and load balancing
- `upstream` blocks and `proxy_pass`
- Load-balancing methods: round robin vs **`least_conn`** (used here) vs `ip_hash`
- Passive health checks: `max_fails`, `fail_timeout`
- Retrying on another backend: `proxy_next_upstream`
- **Keepalive** connections to backends (`keepalive 32`, HTTP/1.1, empty `Connection` header)
- Forwarding headers: `Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`
- Timeouts: connect / read / send
- **WebSockets** through a proxy (`Upgrade` / `Connection` headers)

### 7. Backend protocols (the 5th input)
- **HTTP** and **HTTPS** backends (`proxy_ssl_server_name` for HTTPS)
- **FastCGI / PHP-FPM**: what it is, `fastcgi_pass`, `SCRIPT_FILENAME`, document root
- **gRPC**: what it is, why it needs **HTTP/2**, `grpc_pass`

*Used in:* `setup.py` (`proxy_block`, `fastcgi_block`, `grpc_block`)

### 8. HTTPS / TLS
- What TLS does: encryption and identity
- **Certificates and private keys**, certificate authorities, and why a **self-signed**
  certificate gives a browser warning
- **SAN (Subject Alternative Name)**: one certificate for several domains
- Generating a certificate with **OpenSSL**
- TLS versions (1.2 / 1.3), session cache, HTTP → HTTPS redirect
- **HTTP/2**
- **HSTS**, and why it stays off with a self-signed certificate
- **Let's Encrypt / certbot** for real servers, and DNS **A records**

*Used in:* `certs/`, `setup.py` (`ensure_certificate`, `openssl_config`), `site.conf.tmpl`

### 9. Caching
- HTTP caching headers: `Cache-Control: max-age`, `Expires`
- NGINX `proxy_cache_path`, `keys_zone`, `proxy_cache`, cache key
- Why requests with cookies or `Authorization` headers are never cached
- Serving stale content when the backend fails (`proxy_cache_use_stale`)
- Reading the `X-Cache-Status` header (`HIT` / `MISS` / `BYPASS` / `STALE`)

### 10. Security and hardening
- Security headers: `X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`, HSTS
- `server_tokens off` (hide the NGINX version)
- **Rate limiting**: `limit_req_zone`, `rate`, `burst`, `nodelay`, the leaky-bucket idea
- Upload size limit (`client_max_body_size`)
- IP allow/deny lists and **CIDR** notation (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`)
  for `/nginx_status`

### 11. Performance and observability
- `sendfile`, `tcp_nopush`, `tcp_nodelay`, `keepalive_timeout`
- **gzip** compression and which content types are worth compressing
- Custom `log_format`: request time, upstream time, upstream address, cache status
- `stub_status` (connection counters) and the `/healthz` liveness endpoint

### 12. Docker beyond the basics
- **Bind mounts** (`./nginx/nginx.conf:/etc/nginx/nginx.conf:ro`) vs **named volumes** (`nginx_cache`)
- Read-only mounts (`:ro`)
- Docker networking: containers reach each other by service name (`app1:80`)
- `extra_hosts` and `host.docker.internal:host-gateway` (reaching apps on your PC)
- `restart: unless-stopped`, `depends_on`
- Image tags and why versions are pinned (`nginx:1.27-alpine`, `traefik/whoami:v1.10`)
- What the demo backend `traefik/whoami` does

### 13. Python (standard library only)
To read and change `setup.py`:
- Functions, type hints, dicts, lists, f-strings
- `argparse`: command-line flags (`--from-file`, `--no-start`)
- `json`: saving and loading `nginx-inputs.json`
- `pathlib.Path`: file paths and writing files
- `subprocess`: running `docker` and `openssl` from Python
- `socket`: checking whether a port is busy
- `ipaddress` and `re`: validating IPs, hostnames and ports
- `shutil.which`: finding programs on the PATH
- Templating: `{{NAME}}` placeholders replaced with values (`render`)
- Validating input and showing clear error messages

### 14. Project ideas worth understanding
- **Generated vs source files**: edit `templates/` and `nginx-inputs.json`, never `nginx/` or `docker-compose.yml`
- **Configuration as code**: the same inputs always give the same config
- Idempotent scripts: re-running keeps existing certificates and saved answers

---

## Suggested learning order

1. Command line → Docker basics → run `python setup.py` with the defaults
2. HTTP + curl → work through the checklist in `PLAN.md` section 7
3. NGINX fundamentals → read `templates/nginx.conf.tmpl`
4. Reverse proxy + load balancing → read `templates/site.conf.tmpl`
5. TLS/HTTPS → caching → security → performance and logging
6. Python → read `setup.py` from `main()` downward
7. Optional: a real Linux server, DNS, Let's Encrypt (README section *Moving to a real server*)

## Hands-on checks (you understand it when you can…)

- [ ] Explain why `curl -k https://localhost` returns a different `Hostname` each time
- [ ] Stop `app2` and explain why requests still succeed
- [ ] Explain why `http://localhost` returns `301`
- [ ] Get a `429` by sending many requests quickly
- [ ] Point the gateway at your own app (`python -m http.server 9000` → backend `127.0.0.1:9000`)
- [ ] Change a value in `nginx-inputs.json`, regenerate with `--from-file`, and reload without downtime
- [ ] Read one access-log line and explain every field

## Useful resources

- NGINX docs: https://nginx.org/en/docs/ (especially *Beginner's Guide*, `ngx_http_upstream_module`, `ngx_http_proxy_module`)
- Docker docs: https://docs.docker.com/get-started/ and https://docs.docker.com/compose/
- MDN HTTP guide: https://developer.mozilla.org/en-US/docs/Web/HTTP
- Mozilla SSL config generator: https://ssl-config.mozilla.org/
- Let's Encrypt: https://letsencrypt.org/getting-started/
- Python standard library: https://docs.python.org/3/library/
