# OSM Docker Australia Tile Server – Technical Setup Guide

This document provides a highly technical, step-by-step methodology for deploying an OpenStreetMap (OSM) tile server for Australia using Docker. It covers system preparation, data workflows, container orchestration, advanced database tuning, tile rendering configuration, network access, troubleshooting, and operational recommendations.

---

## Table of Contents

1. System Overview
2. Data Preparation
3. Docker Image Acquisition
4. OSM Data Import Workflow
5. Tile Server Deployment
6. Tile Rendering and Updates
7. Network Configuration
8. PostgreSQL Performance Tuning
9. Renderd / Mapnik Configuration
10. Troubleshooting Matrix
11. Operational Recommendations
12. Verification and Integration
13. References
14. About

---

## 1. System Overview

- **Components:**  
  - OSM Australia `.osm.pbf` extract  
  - PostgreSQL (with PostGIS)  
  - osm2pgsql importer  
  - Mapnik and openstreetmap-carto style  
  - mod_tile, renderd, Apache2  
  - Docker containers for isolation and reproducibility

---

## 2. Data Preparation

Create a persistent directory for OSM data:

```bash
mkdir -p ~/osm-docker/osm-data
mv australia-latest.osm.pbf ~/osm-docker/osm-data/
```

**Best Practice:**  
Keep filenames clean, with no extra dots or spaces. This avoids import and mapping errors.

---

## 3. Docker Image Acquisition

Pull the latest official OSM tile server image:

```bash
docker pull overv/openstreetmap-tile-server:latest
```

---

## 4. OSM Data Import Workflow

Change to your data directory and run the import:

```bash
cd ~/osm-docker/osm-data
sudo docker run -v $(pwd):/data overv/openstreetmap-tile-server:latest import
```

**Technical Details:**

- The container uses `osm2pgsql` to parse and import `.osm.pbf` data into a PostGIS-enabled PostgreSQL database.
- Carto style files and Mapnik configuration are handled internally; Docker moves style files to `/data/style/`.
- Database (`/data/database`) and tile cache (`/data/tiles`) are persisted outside the container.

**Known Issues & Solutions:**

| Issue                                   | Cause                         | Solution                                 |
|------------------------------------------|-------------------------------|------------------------------------------|
| Style folder missing (`/data/style/`)    | Permissions/writable data dir | Ensure data dir is writable              |
| Postgres role "renderer" already exists  | Role reuse                    | Safe to ignore; role reuse is supported  |
| Relation table import failures           | Large relations/dataset       | Clear affected tables/folders, rerun import |

---

## 5. Tile Server Deployment

Start the tile server and expose it on your network:

```bash
sudo docker run -v ~/osm-docker/osm-data:/data -p 8080:80 overv/openstreetmap-tile-server:latest run
```

Tiles are served at:

```
http://<your-server-ip>:8080/tile/{z}/{x}/{y}.png
```

**Integration:**  
Configure web clients (Leaflet, Mapnik) to load tiles using the above URL.

---

## 6. Tile Rendering and Updates

- Tiles are rendered on-demand using Mapnik and cached under `/data/tiles`.
- To update data:
    1. Stop the tile server container.
    2. Replace the `.osm.pbf` file.
    3. Rerun the import.
    4. Restart the container.
- Import completion is tracked by `/data/tiles/planet-import-complete`.

---

## 7. Network Configuration

- **Local access:** `http://<your-server-ip>:8080/tile/{z}/{x}/{y}.png`
- **External access:** Use a reverse proxy (NGINX, Caddy) and configure appropriate port forwarding.
- **Security:** Never expose internal Postgres ports to the public internet.

---

## 8. PostgreSQL Performance Tuning

Apply advanced tuning parameters to optimize database performance (inside container or in `postgresql.conf`):

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

---

## 9. Renderd / Mapnik Configuration

- **Threads:** Set `num_threads=4` or tune according to your CPU core count for optimal render throughput.
- **Fonts:** Ensure required fonts are available at `/usr/share/fonts` for Carto style rendering.

---

## 10. Troubleshooting Matrix

| Issue                                 | Cause                                 | Solution                                                      |
|----------------------------------------|---------------------------------------|---------------------------------------------------------------|
| Blank map in browser                   | Tiles not yet rendered                | Wait for `renderd` to complete initial render; check logs     |
| PostGIS connection errors              | Database not running or import incomplete | Restart container/import, verify DB readiness              |
| `planet-import-complete` missing       | Import incomplete                     | Remove marker file and rerun import                          |
| Slow tile rendering or low detail      | Large dataset or resource constraints | Increase RAM/CPU, use regional extract, tune database params |

---

## 11. Operational Recommendations

- Snapshot `.osm.pbf` sources for historical or rollback purposes.
- Monitor `renderd` logs to visualize tile generation progress.
- Use a tile cache or CDN for scalable, fast distribution across networks.
- For large coverage, consider regional extracts to optimize performance and resource usage.
- Automate data updates with scheduled jobs or CI/CD pipelines.
- Periodically clean old tile cache files if disk space is limited.
- Implement monitoring and alerts for server health and resource usage.

---

## 12. Verification and Integration

### Verify Tile Server Functionality

- Open your browser and access:  
  `http://<your-server-ip>:8080/`
- Test tile endpoints directly:  
  `http://<your-server-ip>:8080/tile/13/4620/6180.png`  
  (Replace with valid `{z}/{x}/{y}` values for Australia.)

### Integrate with Web Mapping Libraries

- Example Leaflet integration:

    ```js
    L.tileLayer('http://<your-server-ip>:8080/tile/{z}/{x}/{y}.png', {
        attribution: 'Map data © OpenStreetMap contributors',
        maxZoom: 18
    }).addTo(map);
    ```

### Monitor Server Health and Logs

- Check Docker container logs for errors or warnings:

    ```bash
    docker logs <container_id>
    ```

- Monitor resource usage: RAM, CPU, disk I/O.
- Ensure `/data/tiles/planet-import-complete` exists (indicates successful import).

### Optimize and Secure

- Review PostgreSQL and renderd/Mapnik performance settings.
- Set up a reverse proxy (NGINX, Caddy) for security, HTTPS, and external access.
- Restrict network access as needed, especially for database ports.

### Update Data and Re-render Tiles

- To update with newer OSM data:
    1. Download the new `.osm.pbf` file.
    2. Stop the tile server container.
    3. Replace the old file in `/osm-data`.
    4. Rerun the import command.
    5. Restart the container.

- Confirm tile updates by checking the client map and logs.

### Advanced: Automation & Maintenance

- Automate data updates using cron jobs or CI/CD pipelines.
- Periodically clean old tiles or cache if disk space is limited.
- Implement monitoring and alerts for server health.

---

## 13. References

- [Overv/openstreetmap-tile-server (GitHub)](https://github.com/Overv/openstreetmap-tile-server)
- [Geofabrik Australia Data](https://download.geofabrik.de/australia-oceania.html)
- [Leaflet Documentation](https://leafletjs.com/)
- [OSM Wiki: Tile server](https://wiki.openstreetmap.org/wiki/Tile_server)
- [openstreetmap-carto style](https://github.com/gravitystorm/openstreetmap-carto)

---

## 14. About

Technical setup and documentation by [Airshiphacker](https://github.com/Airshiphacker).  
Contributions and feedback are welcome.
