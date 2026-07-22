# Deploying to the Linux server

This repo (`frappe-build`) builds a custom Frappe image containing `frappe` +
`payments` + `lms` (see `apps.json`) via GitHub Actions
(`.github/workflows/build-custom-image.yml`) and pushes it to
`ghcr.io/svora2/frappe-build`. The server only needs Docker — it never builds
anything itself.

## One-time GitHub setup (already done)

- `LMS_REPO_TOKEN` secret on this repo — fine-grained PAT, read-only, scoped
  to `svora2/lms` only. Used to clone the private `lms` repo during the build.
- Push to `main` (or run the workflow manually from the Actions tab) to build
  and publish the image.

## On the Linux server

### 1. Install Docker

Follow https://docs.docker.com/engine/install/ for your distro, then the
[post-install steps](https://docs.docker.com/engine/install/linux-postinstall/)
so you don't need `sudo` for every docker command.

### 2. Authenticate to GHCR (image is private)

Create a classic PAT with `read:packages` scope at
github.com/settings/tokens, then:

```shell
echo "<PAT>" | docker login ghcr.io -u svora2 --password-stdin
```

### 3. Clone this repo and configure

```shell
git clone https://github.com/svora2/frappe-build
cd frappe-build
mkdir -p ~/gitops
cp production.env.example ~/gitops/frappelms.env
```

Edit `~/gitops/frappelms.env` and fill in the `CHANGE_ME_*` values:
`DB_PASSWORD`, `NGINX_PROXY_HOSTS` (your domain), `LETSENCRYPT_EMAIL`.

### 4. Generate the compose file

```shell
docker compose --project-name frappelms \
  --env-file ~/gitops/frappelms.env \
  -f compose.yaml \
  -f overrides/compose.mariadb.yaml \
  -f overrides/compose.redis.yaml \
  -f overrides/compose.nginxproxy.yaml \
  -f overrides/compose.nginxproxy-ssl.yaml \
  config > ~/gitops/frappelms.yaml
```

Skip the two `nginxproxy*` override files (and `LETSENCRYPT_EMAIL`/
`NGINX_PROXY_HOSTS`) if you don't have a public domain yet — you can expose
port 8080 directly instead using `overrides/compose.noproxy.yaml`.

### 5. Start the stack

```shell
docker compose --project-name frappelms -f ~/gitops/frappelms.yaml up -d
```

### 6. Create the site

```shell
docker compose --project-name frappelms exec backend \
  bench new-site --mariadb-user-host-login-scope=% \
  --db-root-password <same as DB_PASSWORD> \
  --install-app payments --install-app lms \
  --admin-password <choose one> \
  your-domain.com
```

### 7. Verify

Visit `https://your-domain.com` (or `http://server-ip:8080` if you skipped
the proxy) and log in as `Administrator`.

## Updating later

New commits to `frappe`, `payments`, or `lms` don't redeploy automatically —
re-run the "Build and Push Custom Image" GitHub Action (or push a change to
`apps.json` here), then on the server:

```shell
docker compose --project-name frappelms -f ~/gitops/frappelms.yaml pull
docker compose --project-name frappelms -f ~/gitops/frappelms.yaml up -d
docker compose --project-name frappelms exec backend bench migrate
```
