# Nginx + Nginx UI + Authelia

Docker Compose reverse proxy with two-factor authentication on selected domains.
Nginx and Nginx UI run together in the official Nginx UI image so UI edits and reloads affect the actual proxy. Authelia runs separately with a file-based user directory, SQLite storage, and authenticator-app TOTP. No Docker socket is mounted, and `NGINX_UI_IGNORE_DOCKER_SOCKET=true` disables its setup check. The management proxy forwards HTTP/1.1 Upgrade and Connection headers, disables response buffering, and keeps long-lived WebSocket connections open. Upgrade the Nginx UI container through Docker Compose instead of the UI's Docker-based OTA updater: update `NGINX_UI_IMAGE` in `.env`, then run `docker compose pull nginx && docker compose up -d nginx`.

| Address | Behavior |
| --- | --- |
| `auth.example.com` | Authelia login and 2FA enrollment |
| `secure.example.com` | Demo backend, requires password and TOTP |
| `public.example.com` | Demo backend, no Authelia authentication |
| `http://SERVER-IP:9000` | Nginx UI, accessible directly on any host IPv4 interface |

## Setup

Requires Docker Engine with Compose v2+, Python 3, and OpenSSL. Ports 8080 and 443 must be free. Host port 80 remains available for your existing service. Access application domains directly over HTTPS; configure normal HTTP redirects and HTTP ACME challenge forwarding in the existing port-80 service. These instructions assume Nginx is the internet-facing proxy and terminates TLS directly.

1. Generate your initial Authelia password hash. Enter your password at the interactive prompt:

   ```sh
   docker run --rm -it authelia/authelia:4.39.28 authelia crypto hash generate argon2
   ```

2. Generate configuration, unique secrets, and a 30-day self-signed bootstrap certificate:

   ```sh
   python3 scripts/setup.py --domain example.com --email you@example.com
   ```

   Substitute your actual parent domain and email. Paste only the `$argon2id$...` hash when asked. The initial Authelia username is `admin`. Setup refuses to overwrite existing `data/` or `secrets/`. `.env` controls image versions and published addresses; domain configuration lives in the generated files, not `.env`.

3. Point `auth`, `secure`, and `public` DNS records at your server. For local testing, add these names to your workstation's hosts file with the server IP. Authelia uses HTTPS session cookies across the configured parent domain.

4. Start and validate:

   ```sh
   docker compose config --quiet
   docker compose up -d
   docker compose ps
   docker compose exec nginx nginx -t
   docker compose logs --tail=100 authelia nginx
   ```

5. Open `https://secure.example.com`, sign in as `admin`, and enroll your authenticator. The bootstrap notifier writes enrollment verification links to `data/authelia/notification.txt`; read that private file on the host. The self-signed certificate needs a browser exception on both the application and auth host for this initial test.

6. Open Nginx UI at `http://SERVER-IP:9000` (for example, `http://192.168.225.135:9000`) and create a **separate Nginx UI administrator account**. If requested, retrieve the one-time installation secret:

   ```sh
   docker compose exec nginx cat /etc/nginx-ui/.install_secret
   ```

   `UI_BIND` defaults to `0.0.0.0`, listening on all host IPv4 interfaces. Browse using the server's actual IP address. If your existing `.env` pins `UI_BIND` to a specific IP, change it to `0.0.0.0`. For SSH-tunnel-only access, set `UI_BIND=127.0.0.1`, recreate Nginx with `docker compose up -d nginx`, then run this on your workstation and open `http://localhost:9000`:

   ```sh
   ssh -N -L 9000:127.0.0.1:9000 user@your-server
   ```

## Enable 2FA for another domain

After setup, edit `data/` files directly or use Nginx UI for site edits. `templates/` are only used during initial setup.

1. Copy `data/nginx/sites-available/secure.conf` to a new file, change `server_name` and `$backend`, and enable it in Nginx UI. Alternatively create a relative symlink in `data/nginx/sites-enabled/`, following the existing examples.
2. Keep `authelia-location.conf` included in the server and `authelia-authrequest.conf` included in **every application location** that needs protection. A rule in Authelia alone cannot protect an Nginx location that does not call it. Avoid serving protected content through an Nginx `return` directive, which runs before access checks.
3. Add the hostname to the Authelia rule in `data/authelia/configuration.yml`:

   ```yaml
   access_control:
     default_policy: deny
     rules:
       - domain:
           - secure.example.com
           - app.example.com
         policy: two_factor
   ```

4. Add the hostname to the HTTP redirect server in `default.conf`, configure DNS, and provide a certificate covering the new hostname. The bootstrap certificate covers only the original three names.
5. Validate, restart Authelia, and reload Nginx:

   ```sh
   docker compose exec authelia authelia validate-config --config /config/configuration.yml
   docker compose restart authelia
   docker compose exec nginx nginx -t
   docker compose exec nginx nginx -s reload
   ```

For an unprotected domain, copy `public.conf`; omit both Authelia includes. Unlisted domains that **do** call Authelia are denied by default. Keep the authentication portal itself free of `auth_request` to avoid a login loop.

For unrelated parent domains, add a separate `session.cookies` entry and reachable auth portal for each parent domain, as described in the [Authelia session documentation](https://www.authelia.com/configuration/session/introduction/).

## Connect your applications

Replace `http://demo:80` with the actual upstream URL. For applications in this Compose file, attach them to `proxy` without publishing their application ports. Other Compose projects can join the external network `nginx-webcontrol_proxy`. For services on another host, use their private IP and port and restrict direct access to the proxy host. Remove `demo` and its `depends_on` entry once the examples are replaced.

The shared network is for trusted backend services only. Publicly reachable backend ports would allow clients to bypass Authelia. Forwarded client-IP headers are overwritten at the edge. If another proxy or CDN sits in front of Nginx, configure its exact trusted IP ranges before relying on forwarded addresses. User identity headers are stripped by default; application-specific trusted-header SSO requires additional configuration.

## Production configuration

- Replace `data/certs/fullchain.pem` and `privkey.pem` with a trusted certificate/key covering your enabled hostnames, then reload Nginx. Arrange certificate renewal with your existing ACME client or Nginx UI certificate management. If using UI-managed certificates, update each site's certificate paths accordingly. HTTP ACME challenges are served from `/var/www/.well-known/acme-challenge/`.
- Replace the filesystem notifier with SMTP in `data/authelia/configuration.yml` so enrollment and password reset messages reach users. For example:

  ```yaml
  notifier:
    smtp:
      address: submission://smtp.example.com:587
      username: authelia@example.com
      sender: 'Authelia <authelia@example.com>'
  ```

  Store the password in `secrets/smtp`, add it to Compose `secrets` and the Authelia service's secret list, and set `AUTHELIA_NOTIFIER_SMTP_PASSWORD_FILE: /run/secrets/smtp`. Remove the `filesystem` block. Recreate Authelia with `docker compose up -d authelia` after changing Compose settings.
- Nginx UI is published on all host IPv4 interfaces by default (`UI_BIND=0.0.0.0`) at port 9000. Keep this management endpoint on your trusted network; it has authority to change proxy security rules. Use `UI_BIND=127.0.0.1` for SSH-tunnel-only access. Application routes on ports 8080 and 443 still require their configured domain names; this setting exposes only the management UI by IP.
- Back up `data/`, `secrets/`, and `.env` securely. Stop the stack before a simple file-copy backup so SQLite is consistent. The storage encryption key is required to recover enrolled 2FA devices; do not regenerate it during upgrades.
- The generated runtime files, credentials, certificates, and secrets are gitignored. Versioned image defaults can be changed in `.env`; review upstream releases before updating.

## Verification and maintenance

```sh
# Public demo returns 200; protected demo redirects to auth (302).
# -k is only for the bootstrap self-signed certificate.
curl -kI --resolve public.example.com:443:127.0.0.1 https://public.example.com/
curl -kI --resolve secure.example.com:443:127.0.0.1 https://secure.example.com/

# Validate before reloading edited site configuration.
docker compose exec nginx nginx -t && docker compose exec nginx nginx -s reload

# Stop without deleting persistent configuration or databases.
docker compose down
```

Session state is kept in memory for this single Authelia instance, so restarting it requires users to log in again. SQLite and TOTP enrollment persist in `data/authelia/`. Use Redis and a supported external database if you later need multiple replicas.

References: [Nginx UI Docker installation](https://nginxui.com/guide/getting-started#install-with-docker), [Authelia Nginx integration](https://www.authelia.com/integration/proxies/nginx/).

Validation performed with the pinned images: Compose parsing, generated configuration, Authelia validation, Nginx syntax and container health, public/portal HTTP 200, protected redirect, rejection of spoofed identity headers, inaccessible internal authorization endpoint, and password-only sessions still requiring 2FA. With Authelia stopped, the protected route returned 500 while the public route remained available. Browser-based TOTP enrollment and trusted-certificate issuance require your domain setup and were not exercised.
