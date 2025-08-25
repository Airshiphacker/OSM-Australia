# OpenStreetMap Tile Server Quickstart (Docker, Ubuntu)

A streamlined guide for deploying an OpenStreetMap tile server using Docker on Ubuntu.  
Follow each step and review the notes for common pitfalls.

---

## 1. System Preparation

**Update system and install Docker:**
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose git curl
```
> If you're running on Ubuntu 22.04+, `docker` may be replaced with `docker-ce` from Docker’s official repo for newer versions.

**(Optional) Enable Docker to start on boot:**
```bash
sudo systemctl enable docker
```

---

## 2. Folder Structure

**Create directories for your container and data:**
```bash
mkdir -p ~/osm-docker/osm-data
cd ~/osm-docker
```

---

## 3. Download OSM Data

**Download the OSM extract for your region (example: Australia):**
```bash
wget https://download.geofabrik.de/australia-oceania/australia-latest.osm.pbf -O ~/osm-docker/australia-latest.osm.pbf
```
> Make sure the URL matches your region. [Geofabrik download list](https://download.geofabrik.de/).

---

## 4. Rename & Verify Data File (Critical)

**Rename the downloaded file to a simple name like `region.osm.pbf`:**
```bash
mv ~/osm-docker/australia-latest.osm.pbf ~/osm-docker/region.osm.pbf
```

> ⚠️ The Docker image expects `/region.osm.pbf` as the import source.  
> Renaming avoids errors where Docker can't find the data file.

**Confirm the file exists:**
```bash
ls -lh ~/osm-docker
```

---

## 5. Import OSM Data (One-Time)

**Import into PostgreSQL (this may take hours):**
```bash
sudo docker run -v ~/osm-docker/osm-data:/data \
    -v ~/osm-docker/region.osm.pbf:/data.osm.pbf \
    overv/openstreetmap-tile-server:latest \
    import
```
> The import command mounts your data directory and the OSM file.  
> If you see permission errors, ensure your user is in the `docker` group.

**Successful import creates:**
```
~/osm-docker/osm-data/database/planet-import-complete
```

**If import fails (I/O errors, abrupt shutdown):**
```bash
sudo rm -rf ~/osm-docker/osm-data/database/*
```
Then rerun the import command.

---

## 6. Run the Tile Server

**Start serving tiles:**
```bash
sudo docker run -p 8080:80 \
    -v ~/osm-docker/osm-data:/data \
    overv/openstreetmap-tile-server:latest \
    run
```
> If you get a “port is already allocated” error, check with `sudo lsof -i :8080`.

**Demo page:**  
👉 http://<your-server-ip>:8080

**Example tile:**  
👉 http://<your-server-ip>:8080/tile/0/0/0.png

---

## 7. Common Issues & Fixes

- **Wrong filename:**  
  Always rename your `.osm.pbf` to `region.osm.pbf` and mount as `/data.osm.pbf`.

- **Blank map:**  
  Import not completed. Check logs and confirm `planet-import-complete` exists.

- **Slow at high zoom:**  
  First render is slow; tiles are cached for faster repeated views.

---

## Additional Notes

- **Permissions:** If you get “permission denied” errors, add your user to the `docker` group:
  ```bash
  sudo usermod -aG docker $USER
  newgrp docker
  ```
  Log out/in if necessary.

- **Data updates:** For regularly updated maps, look into [OSM replication](https://wiki.openstreetmap.org/wiki/Replication).

- **File naming:** Extra dots or spaces in filenames may break Docker’s import.

- **Accessing tiles externally:** To allow access from outside your local network, configure firewall and consider a reverse proxy like NGINX or Caddy.

---

**End of Quickstart**
