# openclaw

Installs Docker, clones the OpenClaw repository, builds the Docker image with BuildKit, deploys configuration (env + openclaw.json), patches docker-compose for loopback binding and trusted-proxy auth, and starts services via docker compose.
