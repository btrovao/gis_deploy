### Tutorial: Installing Postgres/PostGIS, DBeaver, GeoServer, QGIS, and Visual Studio Code on Windows and Linux

This guide covers how to install the following applications on **Windows** and **Linux**:  
- **PostgreSQL/PostGIS** (with `postgres_fdw`, `postgis_raster`)  
- **DBeaver**  
- **GeoServer**  
- **QGIS**  
- **Visual Studio Code**

---

## 1. PostgreSQL/PostGIS Installation

### On Windows:

1. **Download PostgreSQL:**
   - Go to the [PostgreSQL website](https://www.postgresql.org/download/windows/) and download the installer.
   
2. **Install PostgreSQL:**
   - Run the installer and follow the instructions. Make sure to select **PostGIS** during the installation process.
   - During the setup, note the port number (default is 5432), username, and password.
   
3. **Enable PostGIS extension:**
   - Open the **pgAdmin** tool or connect via a client.
   - Run the following SQL queries to enable PostGIS:
     ```sql
     CREATE EXTENSION postgis;
     CREATE EXTENSION postgis_raster;
     CREATE EXTENSION postgres_fdw;
     ```

### On Linux (Ubuntu):

1. **Install PostgreSQL:**
   ```bash
   sudo apt update
   sudo apt install postgresql postgresql-contrib
   ```

2. **Install PostGIS:**
   ```bash
   sudo apt install postgis postgresql-14-postgis-3
   ```

3. **Enable PostGIS and Extensions:**
   - Connect to PostgreSQL:
     ```bash
     sudo -i -u postgres
     psql
     ```
   - Enable extensions:
     ```sql
     CREATE EXTENSION postgis;
     CREATE EXTENSION postgis_raster;
     CREATE EXTENSION postgres_fdw;
     ```

---

## 2. DBeaver Installation

### On Windows:

1. **Download DBeaver:**
   - Go to the [DBeaver website](https://dbeaver.io/download/) and download the Windows installer.
   
2. **Install DBeaver:**
   - Run the installer and follow the prompts.

### On Linux (Ubuntu):

1. **Install DBeaver:**
   - Add the DBeaver repository:
     ```bash
     sudo apt update
     sudo apt install dbeaver-ce
     ```

---

## 3. GeoServer Installation

### On Windows:

1. **Download GeoServer:**
   - Go to the [GeoServer website](https://geoserver.org/download/) and download the installer.
   
2. **Install GeoServer:**
   - Run the installer, following the instructions to configure the port and set it up as a service if desired.

3. **Run GeoServer:**
   - Start GeoServer via the shortcut created by the installer.
   - Access GeoServer at `http://localhost:8080/geoserver`.

### On Linux (Ubuntu):

1. **Install Java (GeoServer requires Java):**
   ```bash
   sudo apt update
   sudo apt install openjdk-11-jre
   ```

2. **Download and Extract GeoServer:**
   ```bash
   wget https://sourceforge.net/projects/geoserver/files/GeoServer/latest/geoserver-latest-bin.zip
   unzip geoserver-latest-bin.zip -d /opt/geoserver
   ```

3. **Run GeoServer:**
   ```bash
   cd /opt/geoserver/bin
   ./startup.sh
   ```
   - Access GeoServer via a browser at `http://localhost:8080/geoserver`.

---

## 4. QGIS Installation

### On Windows:

1. **Download QGIS:**
   - Go to the [QGIS website](https://qgis.org/en/site/forusers/download.html) and download the Windows installer.

2. **Install QGIS:**
   - Run the installer and follow the instructions.

### On Linux (Ubuntu):

1. **Install QGIS:**
   ```bash
   sudo apt update
   sudo apt install qgis qgis-plugin-grass
   ```

---

## 5. Visual Studio Code Installation

### On Windows:

1. **Download Visual Studio Code:**
   - Go to the [VS Code website](https://code.visualstudio.com/) and download the installer for Windows.
   
2. **Install Visual Studio Code:**
   - Run the installer and follow the instructions.

### On Linux (Ubuntu):

1. **Install Visual Studio Code:**
   ```bash
   sudo apt update
   sudo apt install code
   ```

---

## Conclusion:

After following these instructions, you should have the following tools installed on either Windows or Linux:
- **PostgreSQL/PostGIS** (with `postgres_fdw` and `postgis_raster`)
- **DBeaver**
- **GeoServer**
- **QGIS**
- **Visual Studio Code**

These tools will enable you to manage, analyze, and visualize spatial data effectively, making them essential for GIS work and geospatial development.
