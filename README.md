# NGINX Gateway

A complete NGINX server from **5 inputs**: domain, listen ports, backend IP(s), backend port(s), backend protocol.
Load balancing, caching, HTTPS, security headers, rate limiting and logging are preconfigured.

See [PLAN.md](PLAN.md) for the design and all default settings.

## Requirements
- Docker Desktop (running)
- Python 3.8+ (standard library only)

## Quick start
```bash
python setup.py
```
Answer the prompts (press **Enter** for defaults). Press Enter at *Backend 1* to use 3 demo backends.

```bash
curl -k https://localhost          # run several times: Hostname rotates app1 / app2 / app3
```

## The inputs
| Prompt | Example | Notes |
|---|---|---|
| Domain | `example.com www.example.com` | several allowed, space-separated |
| HTTP / HTTPS port | `80` / `443` | ports on this PC; the script warns if one is busy |
| Backends | `10.0.0.5:3000` | one per line; `127.0.0.1` / `localhost` is turned into `host.docker.internal` (= this PC) |
| Protocol | `http` | `http`, `https`, `fastcgi` (PHP-FPM, also asks for the document root), `grpc` |

Answers are saved to `nginx-inputs.json` and become the defaults next time.

## Commands
| Command | What it does |
|---|---|
| `python setup.py` | prompts, generates, starts / reloads |
| `python setup.py --from-file` | regenerate from `nginx-inputs.json` (edit it by hand) with no prompts |
| `python setup.py --no-start` | generate files only |
| `docker compose logs -f nginx` | live access log: `rt=` total time, `urt=` backend time, `upstream=` which backend, `cache=` HIT/MISS |
| `docker compose exec nginx nginx -s reload` | reload config with no downtime |
| `docker compose stop app2` | (demo) simulate a backend failure |
| `docker compose down` | stop everything |

Endpoints: `/healthz` (NGINX liveness), `/nginx_status` (connection stats, local only).

## Files
```
setup.py                  generator (run this)
templates/*.tmpl          config templates; {{NAME}} placeholders are filled by setup.py
nginx-inputs.json         your inputs                      (generated)
nginx/nginx.conf          global settings                  (generated)
nginx/conf.d/site.conf    upstream + HTTPS site            (generated)
docker-compose.yml        containers                       (generated)
certs/                    self-signed certificate          (generated, not in git)
```
Don't edit generated files. Change the inputs or the templates and re-run instead.

## Behavior notes
- **Load balancing** uses `least_conn`. A backend that fails 3 times is skipped for 30s, and failed requests are retried on another backend. A backend that silently stops answering costs one request up to 5s (connect timeout) before the retry.
- **Caching** only happens when your app sends `Cache-Control: max-age=...` (or `Expires`). Requests carrying a cookie or `Authorization` header are never cached. Check the `X-Cache-Status` response header.
- **Rate limit** is 10 req/s per IP with a burst of 20; excess requests get `429`.
- **Custom domain locally**: add `127.0.0.1  example.com` to `C:\Windows\System32\drivers\etc\hosts`.

## Moving to a real server with Let's Encrypt
1. Copy the project to a Linux server and point the domain's DNS A record at it.
2. Run `python setup.py` there with your real domain and ports 80/443.
3. Get a certificate (webroot or standalone mode), e.g.
   `certbot certonly --standalone -d example.com` (stop NGINX briefly), then copy
   `fullchain.pem` → `certs/server.crt` and `privkey.pem` → `certs/server.key`.
4. Reload: `docker compose exec nginx nginx -s reload`.
5. Once HTTPS works with the trusted certificate, uncomment the `Strict-Transport-Security`
   line in `templates/site.conf.tmpl` and re-run `python setup.py --from-file`.
