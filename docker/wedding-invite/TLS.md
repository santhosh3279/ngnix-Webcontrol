# Invite proxy TLS

The public Nginx container is `nginx-webcontrol-nginx-1`. Both domains use
`http://192.168.225.189:8080`, including `/assets`, `/files`, and `/socket.io`.
The frontend sets `X-Frappe-Site-Name` to `invite.localhost`; the public proxy
clears any client-supplied value. HTTP redirects preserve the host, path, and query.

## Certificate and renewal

The Let's Encrypt certificate covers `myinvite2u.in` and `*.myinvite2u.in`.
It expires **2027-01-05 09:17:01 UTC**. Renew during December 2026.
Manual DNS-01 issuance does **not** renew unattended. Existing TXT records
cannot be reused for future challenges. Configure a DNS authentication hook or
supported DNS API integration if unattended renewal is required.

Host certificate directory:
`data/certs-terminal/config/live/myinvite2u.in/`

Public Nginx certificate directory:
`/etc/letsencrypt/config/live/myinvite2u.in/`

Run from `/home/cups/ngnix-Webcontrol` when renewal is due:

```sh
docker run --rm -it --name invite-certbot-dns \
  -v /home/cups/ngnix-Webcontrol/data/certs-terminal/config:/etc/letsencrypt \
  -v /home/cups/ngnix-Webcontrol/data/certs-terminal/work:/var/lib/letsencrypt \
  -v /home/cups/ngnix-Webcontrol/data/certs-terminal/logs:/var/log/letsencrypt \
  certbot/certbot:v4.2.0 certonly --manual --preferred-challenges dns \
  --cert-name myinvite2u.in -d myinvite2u.in -d '*.myinvite2u.in'
```

Add each requested TXT value at `_acme-challenge` in GoDaddy. Keep all requested
values present together until validation succeeds, and allow DNS propagation.
After successful renewal, validate before reloading:

```sh
docker exec nginx-webcontrol-nginx-1 nginx -t && docker exec nginx-webcontrol-nginx-1 nginx -s reload
curl --fail --show-error https://myinvite2u.in/api/method/ping
curl --fail --show-error https://keerthi-reddy-santhosh-s.myinvite2u.in/ -o /dev/null
```

The completed issuance's TXT values may be removed after issuance succeeds.

## Deployment record: 2026-10-07

Earlier 502 errors used `192.168.225.135:8080` and reported connection refused.
At inspection, HTTP had already been corrected to `192.168.225.189:8080`.
HTTPS had no matching virtual host and reached the default TLS handshake rejection.

Added TLS to both Invite virtual hosts, validated and reloaded, verified public
HTTPS, then enabled HTTP redirects and validated/reloaded again. Deployment copies
match the active files. Unrelated Nginx configurations are unchanged.

Verification: apex login and published invitation HTTP 200, both ping endpoints
return pong, sampled CSS/JS assets HTTP 200, Socket.IO polling HTTP 200 and WebSocket
upgrades HTTP 101. Public certificate validation succeeded without insecure flags.
Redirects preserve paths and query strings. Four unrelated production hosts retained
their previous responses. No public uploaded files were available in the File table
for a positive download test; uploads share the same catch-all upstream route.

Backup directory: `data/backups/invite-tls-20261007T100537Z/`.
Verification artifacts: `data/invite-tls-staging/`.
