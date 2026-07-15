# crAPI on Rocky Linux (Podman) — Setup Notes

## 1. Install Podman Compose

```bash
sudo dnf install epel-release
sudo dnf install podman-compose
```

## 2. Get crAPI and go to the docker deploy dir

```bash
cd ~/crAPI/deploy/docker
ls
```

You should see `docker-compose.yml`, `docker-compose.minimal.yml`, `.env`, `keys/`, etc.

## 3. Check `.env` BEFORE starting containers

This is the step that bit us. `.env` lives in this same directory (`ls -a` to see it — it's hidden) and it sets defaults used by `docker-compose.yml`. Notably:

```bash
cat .env
```

Look for:

```
LISTEN_IP="127.0.0.1"
```

`docker-compose.yml` publishes ports like:

```yaml
ports:
  - "${LISTEN_IP:-0.0.0.0}:8888:80"
```

The compose file's own *fallback* is `0.0.0.0` (all interfaces), but `.env` explicitly sets `LISTEN_IP=127.0.0.1`, which overrides that fallback and binds every published port to loopback only — even though the compose file looks like it should be open. This is easy to miss because the override lives in `.env`, not in the compose file you'd normally inspect.

### Decide what you want:

- **Localhost only (default, safer):** leave `LISTEN_IP="127.0.0.1"` as-is. Access crAPI via SSH tunnel or from the same host only.
- **Reachable from other machines:** change it to `0.0.0.0` (all interfaces) or a specific host IP.

```bash
sed -i 's/LISTEN_IP="127.0.0.1"/LISTEN_IP="0.0.0.0"/' .env
```

## 4. Start the stack

```bash
sudo podman-compose up -d
```

If you change `.env` **after** already starting containers, recreate them so the new value is picked up:

```bash
sudo podman-compose down
sudo podman-compose up -d
```

## 5. Verify what's actually listening

```bash
sudo podman ps
sudo ss -tulnp
```

Confirm the ports (8888, 8443, 30080, 30443, 8025, etc.) are bound to the address you expect — `0.0.0.0`/`*` for all interfaces, or `127.0.0.1` for loopback-only.

## Checklist for next time

Before running `podman-compose up`:

1. `ls -a` in the compose directory to check for a hidden `.env` file.
2. `cat .env` and note any `LISTEN_IP`, `*_PORT`, or similar override variables.
3. Decide the intended binding (loopback vs. all interfaces) and set `.env` accordingly *before* first boot.
4. After `up -d`, always confirm with `sudo ss -tulnp` — don't assume the compose file's defaults are what's actually running, since `.env` silently wins.
5. Remember: this stack runs on **Podman** (`conmon`, `aardvark-dns` in process list), not Docker — `docker ps` may show nothing or be aliased; use `podman ps` / `podman-compose` commands.
