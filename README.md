## N8N installation on GCP

#### Tech stack:
- GCP
- N8N
- Docker
- Docker-compose
- SSL
- HTTPS
- Domain
- DNS



#### Steps:
1. Create a new project in GCP.
2. Enable Compute Engine API.
3. Create a new VM instance.
   1. In this case configure with the free tier options (https://cloud.google.com/free/docs/free-cloud-features#compute)
      1. 1 non-preemptible e2-micro VM instance per month in one of the following US regions:
         - Oregon: us-west1
         - Iowa: us-central1
         - South Carolina: us-east1
      2. 30 GB-months standard persistent disk
      3. 1 GB of outbound data transfer from North America to all region destinations (excluding China and Australia) per month
   2. Allow http, https and load balancing traffic
4. Press SSH to connect to the VM instance.
5. Commands
   1. Update the system
      ```bash
      sudo apt update
      sudo apt upgrade
      ```
    2. Install Docker
        ```bash
        sudo apt install docker.io
        ```
    3. Package for certificates  
        ```bash
        sudo apt-get install ca-certificates curl gnupg lsb-release
        ```
    4. Create a directory for that
        ```bash
        sudo mkdir -p /etc/apt/keyrings
        ```
    5. Update the system
        ```bash
        sudo apt-get update
        ```
    6. Start Docker
        ```bash
        sudo systemctl start docker
        sudo systemctl enable docker
        ```
    7. Install Docker-compose
        ```bash
        sudo curl -L "https://github.com/docker/compose/releases/download/v2.33.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
        ```
    8. Apply executable permissions to the binary
        ```bash
        sudo chmod +x /usr/local/bin/docker-compose
        ```
    9. Verify the installation
        ```bash
        sudo docker-compose --version
        ```
6. Create docker-compose.yml file
   ```bash
    sudo nano docker-compose.yaml
   ```
    ```yml
    version: "3.7"

    services:
    traefik:
        image: "traefik"
        restart: always
        command:
        - "--api=true"
        - "--api.insecure=true"
        - "--providers.docker=true"
        - "--providers.docker.exposedbydefault=false"
        - "--entrypoints.web.address=:80"
        - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
        - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
        - "--entrypoints.websecure.address=:443"
        - "--certificatesresolvers.mytlschallenge.acme.tlschallenge=true"
        - "--certificatesresolvers.mytlschallenge.acme.email=${SSL_EMAIL}"
        - "--certificatesresolvers.mytlschallenge.acme.storage=/letsencrypt/acme.json"
        ports:
        - "80:80"
        - "443:443"
        volumes:
        - traefik_data:/letsencrypt
        - /var/run/docker.sock:/var/run/docker.sock:ro

    n8n:
        image: docker.n8n.io/n8nio/n8n
        restart: always
        ports:
        - "127.0.0.1:5678:5678"
        labels:
        - traefik.enable=true
        - traefik.http.routers.n8n.rule=Host(`${SUBDOMAIN}.${DOMAIN_NAME}`)
        - traefik.http.routers.n8n.tls=true
        - traefik.http.routers.n8n.entrypoints=web,websecure
        - traefik.http.routers.n8n.tls.certresolver=mytlschallenge
        - traefik.http.middlewares.n8n.headers.SSLRedirect=true
        - traefik.http.middlewares.n8n.headers.STSSeconds=315360000
        - traefik.http.middlewares.n8n.headers.browserXSSFilter=true
        - traefik.http.middlewares.n8n.headers.contentTypeNosniff=true
        - traefik.http.middlewares.n8n.headers.forceSTSHeader=true
        - traefik.http.middlewares.n8n.headers.SSLHost=${DOMAIN_NAME}
        - traefik.http.middlewares.n8n.headers.STSIncludeSubdomains=true
        - traefik.http.middlewares.n8n.headers.STSPreload=true
        - traefik.http.routers.n8n.middlewares=n8n@docker
        environment:
        - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
        - N8N_PORT=5678
        - N8N_PROTOCOL=https
        - NODE_ENV=production
        - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
        - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
        volumes:
        - n8n_data:/home/node/.n8n

    volumes:
    traefik_data:
        external: true
    n8n_data:
        external: true


    ```

7. Create a .env file
    ```bash
    sudo nano .env
    ```

    ```bash
    # The top level domain to serve from
    DOMAIN_NAME=domain.com

    # The subdomain to serve from
    SUBDOMAIN=n8n

    # DOMAIN_NAME and SUBDOMAIN combined decide where n8n will be reachable from
    # above example would result in: https://n8n.example.com

    # Optional timezone to set which gets used by Cron-Node by default
    # If not set New York time will be used
    GENERIC_TIMEZONE=Europe/Berlin

    # The email address to use for the SSL certificate creation
    SSL_EMAIL=email

    ```

8. Create volume for traefik
    ```bash
    sudo docker volume create traefik_data
    ```
9. Create volume for n8n
    ```bash
    sudo docker volume create n8n_data
    ```
10. Setup dns records
    - A record for the subdomain pointing to the VM instance external IP
    - CNAME record for the domain pointing to the subdomain

11. Start the services
    ```bash
    sudo docker-compose up -d
    ```

12. Access n8n
    - https://subdomain.domain.com

### You are good to go
