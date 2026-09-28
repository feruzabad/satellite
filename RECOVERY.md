# Disaster Recovery

Restoring the full stack onto a fresh server after the original is lost.

This only works if the repo survived the server's loss: either
`RESTIC_REPOSITORY` was set to offsite storage (e.g. R2), or it was
local-only and you manually copied `backup/restic-repo/` somewhere else
yourself. A local-only repo that was never copied off the server doesn't
survive losing the server.

## Prerequisites on the new server

- Docker and Task installed, and your private access method set up if it
  needs the host (e.g. VPN installed and authenticated; see README).
- The bootstrap secrets, from a password manager (never only on the server
  itself): `RESTIC_PASSWORD`, and if using offsite storage also
  `RESTIC_REPOSITORY`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`.
  Everything else comes back from the backup.

## Steps

1. Clone the repo:
   ```sh
   git clone <this-repo> satellite && cd satellite
   ```
   If restoring from a manually-copied local repo rather than offsite
   storage, also copy that directory to `satellite/backup/restic-repo/` now.

2. Pre-create the named volumes, empty. MediaFlow is stateless and has none.
   The labels mark them as Compose's own, as if `docker compose up` had
   created them; without them, Compose warns on every start that the volume
   "was not created by Docker Compose":
   ```sh
   for s in caddy; do
     docker volume create \
       --label com.docker.compose.project=$s \
       --label com.docker.compose.volume=data \
       ${s}_data
   done
   ```

3. Restore everything in one shot, using a throwaway container with the same
   mount layout the backup job used, so `restic restore --target /` drops
   files back where they were read from.

   **Offsite repo (R2/S3-compatible):**
   ```sh
   export RESTIC_REPOSITORY=... RESTIC_PASSWORD=... \
          AWS_ACCESS_KEY_ID=... AWS_SECRET_ACCESS_KEY=...

   docker run --rm \
     -e RESTIC_REPOSITORY -e RESTIC_PASSWORD -e AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY \
     -v caddy_data:/mnt/volumes/caddy \
     -v "$(pwd)/restore-env:/mnt/volumes/env" \
     mazzolino/restic:1.8.2 restic restore latest --target /
   ```

   **Manually-copied local repo** (from step 1):
   ```sh
   export RESTIC_PASSWORD=...

   docker run --rm \
     -e RESTIC_REPOSITORY=/mnt/restic -e RESTIC_PASSWORD \
     -v "$(pwd)/backup/restic-repo:/mnt/restic:ro" \
     -v caddy_data:/mnt/volumes/caddy \
     -v "$(pwd)/restore-env:/mnt/volumes/env" \
     mazzolino/restic:1.8.2 restic restore latest --target /
   ```

   Either way, this also drops every service's backed-up `.env` into
   `./restore-env/<service>/.env`.

4. Move the `.env` files into place:
   ```sh
   cp -R restore-env/. . && rm -rf restore-env
   ```

5. Update the value that is tied to the old host: `mediaflow/.env`
   `INTERFACE`, if it was a VPN IP (e.g. `tailscale ip -4` on the new
   server). `127.0.0.1` needs no change.

6. Bring the stack up:
   ```sh
   task up
   ```

7. Re-apply what lives outside the repo, since none of it was backed up:
   - Provider firewall rules (allow 443/tcp, 80/tcp, 443/udp inbound, deny
     the rest). On Oracle Cloud: the subnet's security list or the
     instance's NSG.
   - DNS: if the server's public IP changed, update `MEDIAFLOW_DOMAIN`'s
     A/AAAA records.

8. Verify:
   ```sh
   docker ps
   curl https://<MEDIAFLOW_DOMAIN>/health
   ```
   Confirm MediaFlow's web UI loads through your private access method
   (`http://<INTERFACE>:8888`, or `http://localhost:8888` via an SSH tunnel).
