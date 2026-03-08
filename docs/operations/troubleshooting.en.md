# Troubleshooting

## `host.docker.internal` connection issues

- Symptom: `host.docker.internal` cannot be reached
  - Check: the environment follows the DevContainer / Docker Desktop setup
  - Check: if running directly on the host, the network settings are appropriate

## IPFS restart loop

- Symptom: the IPFS container keeps restarting
  - Action: use `ipfs/kubo:latest` in `ipfs/docker-compose.yaml`

## `/mobile` not reachable from smartphone

- Symptom: `/mobile` is not reachable from the smartphone
  - Check: the frontend was started with `--host 0.0.0.0`
  - Check: both devices are on the same Wi-Fi network
  - Check: the firewall allows `5173`
