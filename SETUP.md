# OpenStreetMap Tile Server (Docker, Ubuntu)

This guide provides a walkthrough for deploying an OpenStreetMap (OSM) tile server using Docker on Ubuntu. It was successfully validated inside a Linux VM and incorporates practical lessons learned during setup using Australia OSM data.

---

## 1. System Preparation

Update your system and install Docker-related dependencies:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose git curl
```

> **Note:** For newer Ubuntu releases, installing `docker-ce` from [Docker’s official repository](https://docs.docker.com/engine/install/ubuntu/) may be preferable.

(Optional) Enable Docker to start on boot:

```bash
sudo systemctl enable docker
```

---

## 2. Directory Structure

Establish directories for container files and imported OSM data:

```bash
mkdir -p ~/osm-docker/osm-data
cd ~/osm-docker
```

---

## 3. Download OSM Data

Acquire the relevant OSM extract for your region. For example, to download Australia:

```bash
wget https://download.geofabrik.de/australia-oceania/australia-latest.osm.pbf -O ~/osm-docker/australia-latest.osm.pbf
```

* Replace the URL with the [Geofabrik](https://download.geofabrik.de/) extract for your region.

---

## 4. Data File Naming (Critical)

**FIXED:** The Docker image requires the data file to be named exactly `region.osm.pbf` **without extra dots, spaces, or typos**.

If the filename is incorrect or misplaced, the import will default to Luxembourg instead of your intended region.

Steps to fix:

```bash
# Rename the downloaded file correctly
mv ~/osm-docker/australia-latest.osm.pbf ~/osm-docker/region.osm.pbf
```

* Confirm the file is correctly named:

```bash
ls -lh ~/osm-docker
```

* Make sure the file path is:

```
~/osm-docker/region.osm.pbf
```

This ensures Docker picks up the correct `.osm.pbf` file for import.

---

## 5. Import OSM Data (One-Time)

**NOTE:** Make sure the filename fix above is completed before running the import.

Import the data into PostgreSQL (may take hours for large regions):

```bash
sudo docker run -v ~/osm-docker/osm-data:/data \
  -v ~/osm-docker/region.osm.pbf:/data.osm.pbf \
  overv/openstreetmap-tile-server:latest import
```

* On successful completion, you’ll see:
  `~/osm-docker/osm-data/database/planet-import-complete`

**If the import fails** (e.g., crash, I/O error, VM shutdown), reset the database and retry:

```bash
sudo rm -rf ~/osm-docker/osm-data/database/*
```

---

## 6. Run the Tile Server

Once import is complete, start the tile server:

```bash
sudo docker run -p 8080:80 -v ~/osm-docker/osm-data:/data overv/openstreetmap-tile-server:latest run
```

* Demo page: `http://<your-server-ip>:8080`
* Example tile: `http://<your-server-ip>:8080/tile/0/0/0.png`

---

## 7. Keeping the Map Updated

The tile server can be kept up-to-date using OSM diffs. You can either update manually or automate the process.

### 7.1. Manual Updates (Recommended)

Run the following command whenever you want to refresh your map data:

```bash
sudo docker run -v ~/osm-docker/osm-data:/data overv/openstreetmap-tile-server:latest update
```

**What this does:**

1. Contacts the OSM replication servers and checks what changes (diffs) have occurred since your last update.
2. Downloads only the differences instead of the full `.osm.pbf` file.
3. Applies these diffs to your PostgreSQL database so that the tile server renders the most current map data.

* If there are no changes, the command finishes quickly.
* If changes exist, they are applied automatically.

This process makes it **fast and efficient**, so for most users, running the update weekly or monthly is sufficient.

### 7.2. Automatic Updates (Optional)

If you want the server to stay continuously up-to-date without manual intervention, you can schedule a cron job. For example, to update hourly:

```bash
0 * * * * docker run -v ~/osm-docker/osm-data:/data overv/openstreetmap-tile-server:latest update
```

* Add this line to your crontab via `crontab -e`.
* This is optional; manual updates using the command above are usually enough for most use cases.

### 7.3. Manual Full Import

If your region changes significantly or you want to update all data, repeat the full import process with the latest `.osm.pbf` file.

---

## 8. Common Issues & Solutions

* **Wrong filename:** Must be `region.osm.pbf`. Skipping this will import Luxembourg by default.
* **Blank map tiles:** Import incomplete—verify `planet-import-complete` in `osm-data/database/`.
* **Port conflicts:** Check with `sudo lsof -i :8080`.
* **Slow initial load:** First render is slow; tiles are cached thereafter.

---

## 9. Additional Notes & Recommendations

* **File naming is critical:** The tile server expects a specific filename for successful import.
* **Permissions:** Add your user to the Docker group to avoid requiring `sudo`:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

* **Performance:** VM import performance is suitable for testing; expect longer times for large regions.
* **Validation:** After setup, the demo page and tile rendering should work as expected.

---

## 10. Production Considerations

For a production deployment, consider:

* **External access:** Configure firewall rules and open necessary ports.
* **Reverse proxy:** Utilize Nginx or Caddy for HTTPS and load balancing.
* **Data persistence & backups:** Regularly back up your PostgreSQL data directory.

---

## 11. OpenStreetMap Tile Usage Policy

If you are running your own OSM tile server, you are not subject to OSM Foundation restrictions. However, if you use tiles from the official OSM servers, you **must** comply with their [Tile Usage Policy](https://operations.osmfoundation.org/policies/tiles/):

* **Do not use OSM’s public tile servers for high-volume production, commercial, or automated use.**
* **Heavy usage is not permitted:** OSM servers are funded by donations and have limited capacity.
* **Bulk download, scraping, or tile mirroring is prohibited.**
* **Caching:** If you use OSM tiles outside your own infrastructure, use local caching to reduce load.
* **Attribution:** Maps must visibly credit “© OpenStreetMap contributors”. See [Attribution Guidelines](https://www.openstreetmap.org/copyright).

> **Recommendation:** For any significant or production use, deploy your own tile server (as described in this guide) or use a commercial tile provider.

---

**References:**

* [OpenStreetMap Tile Server Docker Image](https://github.com/Overv/openstreetmap-tile-server)
* [Geofabrik OSM Data Downloads](https://download.geofabrik.de/)
* [Docker Documentation](https://docs.docker.com/)
* [OSM Tile Usage Policy](https://operations.osmfoundation.org/policies/tiles/)

---

With these steps above, you can deploy a OSM tile server on Ubuntu using Docker, suitable for development, testing, or production environments, and ensure compliance with OSM policies.

** OSM-AtlasCoder **

