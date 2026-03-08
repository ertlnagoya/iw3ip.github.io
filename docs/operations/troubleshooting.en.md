# Troubleshooting

This page groups common failures by symptom and then lists the checks and actions to take.  
Even when several causes are possible, it is usually easiest to start from the closest symptom and work through the checks in order.

## `host.docker.internal` connection issues

- Symptom: `host.docker.internal` cannot be reached
  - Check: the environment follows the DevContainer / Docker Desktop setup
  - Check: if running directly on the host, the network settings are appropriate

## IPFS restart loop

- Symptom: the IPFS container keeps restarting
  - Check: confirm with `docker compose ps` that the IPFS service is the one repeatedly restarting
  - Action: use `ipfs/kubo:latest` in `ipfs/docker-compose.yaml`

## `/mobile` not reachable from smartphone

- Symptom: `/mobile` is not reachable from the smartphone
  - Check: the frontend was started with `--host 0.0.0.0`
  - Check: both devices are on the same Wi-Fi network
  - Check: the firewall allows `5173`

## Product does not appear

- Symptom: product data does not appear in the UI
  - Check: `mediator-owner` was started with `cargo run -- settings/owner_1.yaml`
  - Check: files are being created under `raw_data/output`
  - Action: rerun the flow from sensor input to product generation and confirm intermediate outputs

## Purchase does not succeed

- Symptom: purchase does not succeed
  - Check: MetaMask is connected with the expected account
  - Action: after restarting the local chain, clear stale MetaMask state and reconnect
