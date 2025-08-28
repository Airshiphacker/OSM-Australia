# OpenStreetMap Tile Server (Docker, Ubuntu)

This guide provides a walkthrough for deploying an OpenStreetMap (OSM) tile server using Docker on Ubuntu. It has been updated to ensure full dataset imports and proper style handling, validated inside a Linux VM using Australia OSM data.

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

Add your user to the Docker group to run Docker without `sudo` (recommended):

```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

## 2. Directory Structure

Establish directories for container files and imported OSM data, including the style folder:

```bash
mkdir -p ~/osm-docker/osm-data/style
mkdir -p ~/osm-docker/osm-data/database
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

The Docker image requires the data file to be named exactly `region.osm.pbf` **without extra dots, spaces, or typos**.

Rename the downloaded file correctly:

```bash
mv ~/osm-docker/australia-latest.osm.pbf ~/osm-docker/region.osm.pbf
```

* Confirm the file is correctly named:

```bash
ls -lh ~/osm-docker
```

This ensures Docker picks up the correct `.osm.pbf` file for import.

---

## 5. Import OSM Data (One-Time)

**NOTE:** Ensure the filename and directory structure above are correct before running the import.

**Updated Import Command:**

```bash
sudo docker run -v ~/osm-docker/osm-data:/data \
  overv/openstreetmap-tile-server:latest import
```

* Only mount the `osm-data` folder, not the `.osm.pbf` file separately.
* On successful completion, you’ll see:
  `~/osm-docker/osm-data/database/planet-import-complete`

If the import fails (e.g., crash, I/O error, VM shutdown), reset the database and retry:

```bash
sudo rm -rf ~/osm-docker/osm-data/database/*
```

---

## 6. Run the Tile Server

Start the tile server:

```bash
sudo docker run -p 8080:80 -v ~/osm-docker/osm-data:/data overv/openstreetmap-tile-server:latest run
```

* Demo page: `http://<your-server-ip>:8080`
* Example tile: `http://<your-server-ip>:8080/tile/0/0/0.png`

---

## 7. Keeping the Map Updated

### 7.1. Manual Updates (Recommended)

```bash
sudo docker run -v ~/osm-docker/osm-data:/data overv/openstreetmap-tile-server:latest update
```

This command downloads OSM diffs and applies them to your database efficiently.

### 7.2. Automatic Updates (Optional)

Schedule a cron job to update periodically:

```bash
0 * * * * docker run -v ~/osm-docker/osm-data:/data overv/openstreetmap-tile-server:latest update
```

### 7.3. Manual Full Import

If your region changes significantly, repeat the full import process with the latest `.osm.pbf` file.

---

## 8. Common Issues & Solutions

* **Wrong filename:** Must be `region.osm.pbf`; otherwise, Luxembourg is imported.
* **Blank map tiles:** Import incomplete—verify `planet-import-complete` in `osm-data/database/`.
* **Port conflicts:** Check with `sudo lsof -i :8080`.
* **Slow initial load:** First render is slow; tiles are cached thereafter.

---

## 9. Additional Notes & Recommendations

* Permissions: Add your user to the Docker group to avoid `sudo`.
* Performance: VM import performance is suitable for testing; large regions may take longer.
* Validation: Ensure demo page and tiles render correctly.

---

## 10. Production Considerations

* Configure firewall and open ports for external access.
* Use a reverse proxy like Nginx or Caddy for HTTPS.
* Regularly back up your PostgreSQL data.

---

## 11. OpenStreetMap Tile Usage Policy

Follow the [OSM Tile Usage Policy](https://operations.osmfoundation.org/policies/tiles/) if using official OSM servers. For your own server:

* You are free to use tiles without restriction.
* Ensure attribution: “© OpenStreetMap contributors”.

---

**References:**

* [OpenStreetMap Tile Server Docker Image](https://github.com/Overv/openstreetmap-tile-server)
* [Geofabrik OSM Data Downloads](https://download.geofabrik.de/)
* [Docker Documentation](https://docs.docker.com/)
* [OSM Tile Usage Policy](https://operations.osmfoundation.org/policies/tiles/)

**OSM-AtlasCoder**
