# Easy Deployment Instructions via Docker Compose

## Prerequisites

- Docker and Docker Compose installed on your machine

  ```bash
  # For Ubuntu/Debian systems

  # Check docker installation
  docker --version

  # ------------------- If Docker is not installed, run the following commands -------------------

  # Update package lists
  sudo apt update
  sudo apt upgrade -y

  # Install Docker
  curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh

  # First, ensure the Docker background service is actively running and enabled to start on boot
  sudo systemctl status docker

  # Test with the "Hello World" image
  sudo docker run hello-world

  ```

  > If your installation was successful, Docker will download the test image and print a welcome message that starts with: "Hello from Docker! This message shows that your installation appears to be working correctly."

- (Optional) Add the user to the `sudo` group to run Docker commands without `sudo`:
  ```bash
  sudo usermod -aG sudo $USER
  ```
  > After running this command, log out and log back in for the changes to take effect.

## Step 01: Create the GitHub Deploy Key

- Generate the SSH key:

  ```bash
  ssh-keygen -t ed25519 -C "crisislab-server"
  ```

  > When prompted for a file to save the key, you can specify a custom path (e.g., `~/.ssh/crisislab_server_key`) or press Enter to use the default location (`~/.ssh/id_ed25519`).

- Copy the public key to your clipboard:

  ```bash
  cat ~/.ssh/id_ed25519.pub
  ```

  > This will display the public key in the terminal. Copy the entire output, which should start with `ssh-ed25519` and end with `crisislab-server`.

- Add the public key as a deploy key in your GitHub repository:
  1. Go to your GitHub repository.
  2. Click on "Settings" > "Deploy keys" > "Add deploy key".
  3. Paste the public key into the "Key" field and give it a title (e.g., "CrisisLab Server Key").
  4. Check the "Allow write access" box if you want the server to have write permissions (optional).
  5. Click "Add key" to save the deploy key.

## Step 02: Pull the code and set up the environment

- Clone the repository:

  ```bash
  git clone https://github.com/Bimsara-Janakantha/EEW_Sensor.git
  cd EEW_Sensor
  ```

- Create a `.env` file configured with your database credentials and other settings (see the .env_example file). For the demonstration purposes, you can use the default credentials provided in the .env_example file, but make sure to copy them to a new `.env` file and adjust as needed.

## Step 03: Generate JWKs for JWT signing

- Generate a new pair of JWKs for signing and verifying JWTs. The server requires a public/private JSON Web Key (JWK) pair. If you don't have Deno installed locally, you can use Docker to run the script and generate them.

  ```bash
  docker run --rm -v ${PWD}:/app -w /app denoland/deno:alpine run /app/scripts/generate-jwk.js
  ```

- The script will output a PRIVATE_JWK and a PUBLIC_JWK.
- Copy these two values and paste them into your .env file, replacing the placeholder {...} values on lines 11 and 12.

## Step 04: Start the server with Docker Compose

- Build the ingest-deno image:

  ```bash
  docker build -t ingest-deno .
  ```

- Start the server using Docker Compose:

  ```bash
  docker-compose up -d
  ```

- Verify that the server is running:
  ```bash
  docker-compose ps
  ```
  > You should see both the timescaledb and ingest-server containers showing as "Up".
