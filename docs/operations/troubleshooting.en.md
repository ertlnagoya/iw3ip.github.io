# Troubleshooting

This page groups common failures by symptom and then lists the checks and actions to take.  
Even when several causes are possible, it is usually easiest to start from the closest symptom and work through the checks in order.

## `host.docker.internal` connection issues

- Symptom: the mediators (`mediator-owner` / `mediator-buyer`) or the frontend cannot reach `host.docker.internal` and DNS resolution fails
  - Example log: `ConnectError("dns error", ... "failed to lookup address information: nodename nor servname provided, or not known")`
  - Example (frontend): `JsonRpcProvider failed to detect network ... (perhaps the URL is wrong or the node is not started)` printed every second
  - Cause: `host.docker.internal` is a special hostname that Docker Desktop provides **only inside containers**. When you run a process **outside a container (natively on the host OS)** with `cargo run` or `npm run dev`, that name cannot be resolved.

### Check

- Confirm the node itself is alive by running this on the host:

  ```bash
  curl http://localhost:8545
  ```

  If it returns `{"jsonrpc":"2.0", ... "error": ... "Parse error: Unexpected end of JSON input"}`, the **node is reachable and healthy** — this is the normal JSON-RPC response to a GET with an empty body, not an error.

### Fix A — keep running natively (the `cargo run` steps in quickstart)

Replace `host.docker.internal` with `localhost` in the settings files.

- `mediator-owner/settings/owner_1.yaml`:

  ```yaml
  api_url: "http://localhost:3000"
  rpc_url: "http://localhost:8545"
  ```

- Apply the same change to the corresponding `mediator-buyer` settings and to the frontend (`iot-market-ui`) RPC endpoint configuration. If you cannot find where the endpoint is set, grep the repository for `host.docker.internal`.
- Restart each process after editing.

### Fix B — run inside containers

Run the mediators and frontend on the DevContainer / Docker side, where `host.docker.internal` resolves as intended. Choose this if you prefer not to change the settings.

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
