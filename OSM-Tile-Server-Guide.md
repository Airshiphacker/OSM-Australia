# OpenStreetMap Tile Server Deployment Guide

This guide outlines a refined process for deploying an OpenStreetMap (OSM) tile server on a clean Ubuntu system using Docker. Follow each section carefully for a reliable and maintainable installation.

---

## 1. Prepare Your VM

- **Recommended:** Ubuntu 20.04 or newer, freshly installed.
- **Update and install dependencies:**
  ```bash
  sudo apt update && sudo apt upgrade -y
  sudo apt install -y docker.io docker-compose git wget unzip
  ```
- **Add your user to the Docker group:**
  ```bash
  sudo usermod -aG docker $USER
  newgrp docker
  ```
  > Log out and back in if you see permission errors using Docker.

---

## 2. Directory Structure

- **Create main directories:**
  ```bash
  mkdir -p ~/osm-docker/osm-data
  cd ~/osm-docker
  ```
- **Recommended folder layout:**
  ```
  osm-docker/
   ├─ osm-data/           # Stores PBF, style, DB, tiles
   └─ docker-compose.yml  # (Optional, for orchestration)
  ```

---

## 3. Download Regional OSM Extract

- **Download the latest Australia extract:**
  ```bash
  cd ~/osm-docker/osm-data
  wget https://download.geofabrik.de/australia-oceania-latest.osm.pbf
  ```
- **Rename the file (prevents import errors):**
  ```bash
  mv australia-oceania-latest.osm.pbf australia-latest.osm.pbf
  ```
  > Ensure the filename is simple, without extra dots or spaces.

---

## 4. Prepare Carto Style

- **Clone the Carto style repository:**
  ```bash
  cd ~/osm-docker/osm-data
  git clone https://github.com/gravitystorm/openstreetmap-carto.git style
  ```
- **Compile the style XML (requires `carto`):**
  ```bash
  cd style
  carto project.mml > mapnik.xml
  ```
  > If you do not have `carto` installed locally, use a Docker image that includes it or refer to the Carto documentation.

---

## 5. Import OSM Data

- **Mount the data directory and run the import:**
  ```bash
  cd ~/osm-docker/osm-data
  sudo docker run -v $(pwd):/data \
    overv/openstreetmap-tile-server:latest \
    import
  ```
- **Troubleshooting import issues:**
  - `ls: cannot access '/data/style/': No such file or directory`  
    → Ensure style files exist at `/data/style/`.
  - `mv: target '/data/style/' is not a directory`  
    → Confirm `/data/style` exists and is a directory.
  - **IO interruptions:**  
    Delete `/data/database/PG_VERSION` and `/nodes/flat_nodes.bin` to rerun import.
  - **Partial imports:**  
    Delete `/data/tiles/planet-import-complete` if stuck or incomplete.

---

## 6. Postgres Configuration (Advanced/Optional)

- The Docker container handles the database by default.
- **To customize Postgres (for large imports):**
  ```bash
  sudo nano /etc/postgresql/15/main/postgresql.conf
  ```
  Adjust these parameters as needed:
  ```
  max_connections = 250
  temp_buffers = 32MB
  work_mem = 128MB
  wal_buffers = 1MB
  wal_writer_delay = 500ms
  commit_delay = 10000
  max_wal_size = 2880MB
  random_page_cost = 1.1
  autovacuum_vacuum_scale_factor = 0.05
  autovacuum_analyze_scale_factor = 0.02
  listen_addresses = '*'
  ```
- **Ensure renderer role exists:**
  ```bash
  sudo -u postgres createuser renderer
  sudo -u postgres createdb -O renderer gis
  ```
  > If the role already exists, skip this step.

---

## 7. Run the Tile Server

- **Start the tile server:**
  ```bash
  sudo docker run -v ~/osm-docker/osm-data:/data \
    -p 8080:80 \
    overv/openstreetmap-tile-server:latest \
    run
  ```
- **Common runtime issues:**
  - **Port is already allocated:**  
    Another container may be using port 8080. Check with:
    ```bash
    sudo lsof -i :8080
    ```
  - **Blank static maps:**  
    - Verify `/data/database/planet-import-complete` exists.
    - Confirm database connection (see container logs).
    - Ensure VM IP matches your tile URL.

---

## 8. Verify Tile Rendering

- **Test with a static tile URL:**
  ```
  http://<YOUR_VM_IP>:8080/tile/0/0/0.png
  ```
- **Interactive map:**  
  Open a local HTML template (e.g., `map.html`) in your browser and configure it to use your tile server endpoint.

---

## 9. (Optional) Auto-Update & Planet Import

- **For automatic updates:**
  ```bash
  docker exec -it <container_name> bash
  osmosis --read-replication-interval ...
  ```
- **Manual completion signal:**
  ```bash
  touch /data/database/planet-import-complete
  ```

---

## 10. Operational Notes & Best Practices

- **File naming for extracts is critical:**  
  Extra dots or non-standard names may break Docker import.
- **I/O errors during import:**  
  May leave the database in a partial state. Remove `/data/database/PG_VERSION` to restart.
- **Tile rendering:**  
  Requires Postgres running inside the container.
- **Zoom detail:**  
  Depends on VM hardware and Docker resource allocation.
- **Access outside the LAN:**  
  Requires a reverse proxy or port forwarding.

---

## 11. Accessing Your Map

- **Local network access:**  
  ```
  http://<YOUR_VM_IP>:8080/tile/{z}/{x}/{y}.png
  ```
- **Web access:**  
  This requires a reverse proxy (e.g., NGINX or Caddy) or NAT port forwarding.  
  *Do not expose the Postgres database directly to the internet.*

---

## 12. Reverse Proxy Considerations (Optional)

For secure and scalable public access, it is recommended to use a reverse proxy such as **NGINX** or **Caddy**.  
A reverse proxy can:

- Terminate HTTPS and handle certificates.
- Provide access control and logging.
- Forward requests from standard ports (80/443) to your tile server (8080).
- Protect the backend database and services.

Refer to official [NGINX documentation](https://nginx.org/en/docs/) or [Caddy documentation](https://caddyserver.com/docs/) for proxy configuration examples.

---

**End of Guide**
