# 🚀 Caldera Docker Installation & Usage Guide

## 🐳 Docker Installation

### Local Build

Clone the Caldera repository and build the Docker image locally:

```bash
git clone https://github.com/mitre/caldera.git --recursive
cd caldera
docker build --build-arg VARIANT=full -t caldera .
docker run -it -p 8888:8888 caldera
```

- Adjust the port forwarding (`-p`) and build args (`--build-arg`) as needed to expose ports or change the Caldera variant.
- The ports you expose depend on which contacts you plan to use (see `Dockerfile` and `docker-compose.yml` for reference).

### Pre-Built Image (from GitHub Container Registry)

```bash
docker run -p 8888:8888 ghcr.io/mitre/caldera:latest
```

> **Note:** This container may be slightly outdated. Building the container yourself is recommended for the latest features and fixes.

---

## 🛑 Graceful Container Shutdown

1. Find the container ID for your running Caldera container:
	```bash
	docker ps
	```
2. Stop the container:
	```bash
	docker stop <container ID>
	```

---

## 🏷️ Caldera Docker Variants

- **full**: Includes all files, suitable for offline operation.
- **slim**: Excludes files for the emu and atomic plugins (downloaded on-demand if enabled). Slim images on GHCR are prefixed with `slim`.

---

## 📦 Docker Container Notes

- The Caldera container auto-generates keys, usernames, and passwords on first start.
- To override the default config or avoid auto-generated credentials, bind-mount your own config file:
  ```bash
  -v <your_path>/conf.yml:/usr/src/app/conf/local.yml
  ```
- Data is ephemeral by default. To persist data, use Docker volumes or bind mounts:
  ```bash
  -v <path_to_your_data_or_volume_name>:/usr/src/app/data/
  ```
  - Ensure the directory structure matches the `data/` directory in the Caldera repo.
  - Persist your config file to avoid issues with encryption keys.
- The builder plugin does **not** work within Docker.
- To modify atomic plugin data, clone the Atomic Red Team repo outside the container and bind-mount it to `/usr/src/app/plugins/atomic/data/atomic-red-team`.
- To modify emu plugin data, clone the adversary_emulation_library repo and bind-mount it to `/usr/src/app/plugins/emu/data/adversary-emulation-plans`.
- **Security:** Review security best practices before deploying Caldera in production environments.

---

## 🖥️ User Interface Development

If you plan to develop the Caldera UI, follow these additional steps:

### Requirements
- NodeJS (v16+ recommended)

### Setup
1. Add the Magma submodule (if not already present):
	```bash
	git submodule add https://github.com/mitre/magma
	```
2. Install NodeJS dependencies:
	```bash
	cd plugins/magma && npm install && cd ..
	```
3. Start the Caldera server with the UI dev flag:
	```bash
	python3 server.py --uidev localhost
	```

- The Caldera server will be available at [http://localhost:8888](http://localhost:8888).
- The hot-reloading VueJS front-end will be available at [http://localhost:3000](http://localhost:3000).
- Both server and front-end logs will display in the terminal you launched the server from.

---

## 🗄️ Creating Persistent Config & Data Before First Run

If you want stable credentials and encryption keys from the start, set up persistent folders and configuration before running Caldera for the first time.

### 1. Create Persistent Folders

```bash
mkdir -p ~/caldera-persist/data
mkdir -p ~/caldera-persist/config
mkdir -p ~/caldera-persist/exfil
mkdir -p ~/caldera-persist/ftp_dir
```

You can mount these folders to the appropriate paths in your Caldera container for persistent exfiltration and FTP data storage:

```bash

docker run -d --restart unless-stopped --name caldera \
  -p 8888:8888 \
  -v ~/caldera-persist/data:/usr/src/app/data \
  -v ~/caldera-persist/config/local.yml:/usr/src/app/conf/local.yml \
  -v ~/caldera-persist/exfil:/tmp/caldera \
  -v ~/caldera-persist/ftp_dir:/usr/src/app/ftp_dir \
  caldera

```

### 2. Copy Data Skeleton from the Image

This ensures all required subfolders exist exactly as Caldera expects:

```bash
docker create --name caldera-tmp caldera:local
docker cp caldera-tmp:/usr/src/app/data ~/caldera-persist/
# Now you have ~/caldera-persist/data/* with the right structure
```

### 3. Get a Starting Config File

If you cloned the repo:

```bash
cp ./conf/default.yml ~/caldera-persist/config/local.yml
```

Or pull it from the image:

```bash
docker cp caldera-tmp:/usr/src/app/conf/default.yml ~/caldera-persist/config/local.yml
```

### 4. Remove the Temporary Container

```bash
docker rm caldera-tmp
```

### 5. (Optional) Edit Your Config

You can now edit `~/caldera-persist/config/local.yml` to set fixed credentials, keys, or other settings before first launch.

---

## 🔑 Caldera Secrets Explained

- **UI passwords (users):**
  - These are what you type into the web UI to log in (e.g., `admin`, `red`, `blue`).

- **API keys (`api_key_red`, `api_key_blue`):**
  - Used by scripts or tools to call Caldera’s REST API without a username/password.
  - **Agents do NOT use these.** Agents authenticate using their contact/channel config (such as Sandcat over HTTP/WS/TCP) and are managed by the server, not via these API keys.

- **Crypto keys (`encryption_key`, `crypt_salt`):**
  - Used to encrypt Caldera data.
  - **Set these once at the start.** Changing them later can make old data unreadable.

- **Contact passwords (e.g., `tunnel.ssh.user_password`, `ftp.pword`):**
  - Credentials used by those specific contacts only (e.g., for SSH tunnels or FTP access).

