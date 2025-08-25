# OpenStreetMap Tile Server Deployment Guide

This document provides a clear, step-by-step procedure for deploying an OpenStreetMap (OSM) tile server using Docker on a fresh Ubuntu VM. Follow each section in order for a successful setup and reliable operation.

---

## 1. VM Preparation

- Start with a **fresh Ubuntu VM** (recommended: Ubuntu 22.04 or later).
- Update all packages:
  ```bash
  sudo apt update && sudo apt upgrade -y
  ```
- Install required software:
  ```bash
  sudo apt install -y docker.io docker-compose postgresql postgis git wget unzip
  ```
- Add your user to the Docker group (replace `$USER` as needed):
  ```bash
  sudo usermod -aG docker $USER
  ```
- **Log out and log back in** to apply Docker group changes.

---

## 2. Directory Structure

- Create directories for OSM data and backups:
  ```bash
  mkdir -p ~/osm-docker/osm-data
  mkdir -p ~/osm-docker/osm-backup
  ```

---

## 3. Obtain OSM Data

- Navigate to the data directory:
  ```bash
  cd ~/osm-docker/osm-data
  ```
- Download the Australia OSM extract:
  ```bash
  wget https://download.geofabrik.de/australia-oceania/australia-latest.osm.pbf
  ```
- **Verify the filename:**  
  It should be `australia-latest.osm.pbf` (no extra dots or spaces).

---

## 4. Acquire Docker Image

- Pull the latest OpenStreetMap tile server image:
  ```bash
  docker pull overv/openstreetmap-tile-server:latest
  ```

---

## 5. Import OSM Data

- Ensure you are in the correct directory:
  ```bash
  cd ~/osm-docker/osm-data
  ```
- Run the import:
  ```bash
  sudo docker run -v $(pwd):/data overv/openstreetmap-tile-server:latest import
  ```
- **Notes:**
  - The Docker image will handle style folder setup automatically.
  - Confirm `/osm-data` is writable.

- **If import fails or is incomplete:**
  ```bash
  rm -rf /data/database/*
  rm -rf /data/tiles/*
  ```
  Then rerun the import command.

---

## 6. Verify Import Completion

- Check for the import completion file:
  ```bash
  ls ~/osm-docker/osm-data/tiles/planet-import-complete
  ```
- Inspect database tables:
  ```bash
  sudo docker run -it -v $(pwd):/data --rm overv/openstreetmap-tile-server:latest psql -U renderer -d gis -c "\dt"
  ```

---

## 7. Run the Tile Server

- Launch the tile server:
  ```bash
  sudo docker run -v ~/osm-docker/osm-data:/data -p 8080:80 overv/openstreetmap-tile-server:latest run
  ```
- **Verify operation:**
  - Access: `http://<your_vm_ip>:8080/tile/0/0/0.png`
  - You may use a local web client pointing to the above tiles URL.

---

## 8. Performance Tuning

- **Recommended VM specifications:**  
  Minimum 8–16 GB RAM and multiple CPU cores.
- **Optional Postgres tuning** (performed inside the container for advanced deployments):
  ```
  max_connections = 250
  temp_buffers = 32MB
  work_mem = 128MB
  wal_buffers = 1024kB
  wal_writer_delay = 500ms
  commit_delay = 10000
  max_wal_size = 2880MB
  random_page_cost = 1.1
  autovacuum_vacuum_scale_factor = 0.05
  autovacuum_analyze_scale_factor = 0.02
  listen_addresses = '*'
  autovacuum = on
  ```
- For `renderd`, set `num_threads=4` or equal to the number of CPU cores.

---

## 9. Network Access

- **Local network access:**  
  Use `http://<your_vm_ip>:8080/tile/...`
- **External access:**  
  For production deployments, it is recommended to place the tile server behind a reverse proxy (such as NGINX or Caddy) or use a VPN.  
  **Never expose the Postgres database directly to the internet.**

---

## 10. Updating Map Data

- When a new `.osm.pbf` extract is released:
  1. Stop the running tile server container.
  2. Replace the `.osm.pbf` file with the new version.
  3. Clear the database and tile cache:
      ```bash
      rm -rf ~/osm-docker/osm-data/database/*
      rm -rf ~/osm-docker/osm-data/tiles/*
      ```
  4. Re-import the data (see step 5).
  5. Restart the tile server (see step 7).

---

## 11. Logs & Monitoring

- Tile rendering logs are available via the container’s `stdout`.
- Track import progress with `/data/tiles/planet-import-complete`.
- `renderd` errors typically indicate missing tiles or database connection issues.

---

## 12. Backup & Recovery

- **Database backup:**  
  Periodically back up `~/osm-docker/osm-data/database`
- **Tile cache backup:**  
  Back up `~/osm-docker/osm-data/tiles` for faster recovery.

---

## 13. Troubleshooting

| Issue                         | Cause                                | Solution                                        |
|-------------------------------|--------------------------------------|-------------------------------------------------|
| Blank map                     | Tiles not rendered yet               | Wait or check renderd logs                      |
| PostGIS socket failed         | DB not running / import incomplete   | Re-run import                                   |
| planet-import-complete missing| Partial import                       | Clear DB/tiles and re-import                    |
| Slow zoom                     | Large PBF / insufficient RAM         | Increase VM resources or use smaller extract    |
| Docker port bind fails        | Port 8080 already in use             | Stop conflicting container and rerun            |

---

## 14. Reverse Proxy (Optional)

For public deployments, consider using a reverse proxy to improve security, performance, and flexibility.  
Popular choices include **NGINX** and **Caddy**.  
A reverse proxy can handle HTTPS, rate limiting, and access control, while forwarding requests to your tile server running on port 8080.

*Configuration of a reverse proxy is outside the scope of this guide, but official documentation for NGINX and Caddy provides clear instructions for proxying HTTP services.*

---

**End of guide**
