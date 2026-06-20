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

## Events are rejected with `no_matching_consent`

- Symptom: in the HA Demo Simulator (or similar), some datasets never appear in `/platform/ingest`. In `audit/logs` they show `action: deny` / `reason: no_matching_consent` / `subject_did: "unknown"`
  - Example log:

    ```json
    {"action":"deny","subject_did":"unknown","dataset_id":"home/event/possible_littering","purpose":"research","reason":"no_matching_consent", ...}
    ```

  - Most likely cause: **an expired consent VC**. If the `valid_to` of a consent file (`examples/ha_demo/consent_*.json`) is in the past, the consent no longer matches at the current time and the event is denied even after registration. The bundled sample files use fixed dates, so they expire as time passes.

### Check

- Read the audit log first (in zsh, quote the URL because `?` is treated as a glob):

  ```bash
  curl 'http://localhost:8080/audit/logs?limit=20'
  ```

- Open the consent file for the denied `dataset_id` and confirm that **today's date** falls within `valid_from`–`valid_to`. If `valid_to` is in the past, the consent has expired.

### Fix

- Extend `valid_to` to a sufficiently future date and re-register. Example to bump all expired entries at once (macOS `sed` needs `-i ''`):

  ```bash
  sed -i '' 's/"valid_to": "2026-05-01T00:00:00Z"/"valid_to": "2027-12-31T23:59:59Z"/' examples/ha_demo/consent_*.json
  ```

- After editing, re-register the consents (confirm each response contains `"status":"stored"`), re-send the events, then check `/platform/ingest` again.
- Existing `deny` entries remain in the log as history. Always verify with **a new event sent after the consent was registered**.
- Permanent fix: in the course repository, set the consent `valid_to` to a long-lived date (or generate it as "now + 1 year" at runtime).

## Merchandise page returns 500 (iot-market-ui endpoint)

- Symptom: the list page (`/`) opens, but an individual merchandise page `http://localhost:5173/merchandise/<address>` returns **HTTP 500**
  - Cause: the frontend (`iot-market-ui`) reads its RPC and publisher endpoints from `iot-market-ui/.env.local`. If that endpoint **cannot reach the running Hardhat node** (someone else's IP, a wrong port such as `8546`, or a host that is not up), the merchandise page fails to fetch data and returns 500. The list page has a timeout and still renders, so the failure first surfaces on the detail page.

### Check and fix

- Open `iot-market-ui/.env.local` and confirm `VITE_RPC_URL` points at the running node (default `http://127.0.0.1:8545`). If the file is missing, copy the template.

  ```bash
  cd iot-market-ui
  cp .env.example .env.local
  ```

  ```
  VITE_RPC_URL=http://127.0.0.1:8545
  VITE_PUBLISHER_URL=http://127.0.0.1:8080
  ```

- For phone access, replace `127.0.0.1` with the PC's LAN IP (e.g. `http://192.168.1.20:8545`).
- Restart the frontend (`npm run dev`) after editing.
- Note: if a contract-address mismatch is suspected, stop and restart the Hardhat node (`npx hardhat node`) and redeploy so the addresses align with the defaults. If they still differ, update `iot-market-ui/src/lib/contracts/IoTMarket.ts` and `PubKey.ts` to the deployed addresses.

## Cannot add the network to MetaMask

- Symptom: adding a `http://...:8545` network to MetaMask (on a phone or PC) is rejected no matter how many times you try
  - Cause: MetaMask validates the network by connecting to the RPC when you add it. If the endpoint is unreachable, it is rejected. From a phone in particular, the default `npx hardhat node` only listens on `127.0.0.1`, so the phone cannot reach it.

### Check and fix

- Restart the node exposed on the LAN.

  ```bash
  npx hardhat node --hostname 0.0.0.0
  ```

- Use the **PC's LAN IP** for the RPC URL in MetaMask (e.g. `http://192.168.1.20:8545`), with chain ID `31337`.
- Confirm the PC firewall allows `8545` and that the PC and phone are on the same Wi-Fi.
- If you use the frontend from the phone, set `iot-market-ui/.env.local` to the same LAN IP (see the previous item).
