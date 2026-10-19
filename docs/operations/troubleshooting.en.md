# Troubleshooting

This page groups common failures by symptom and then lists the checks and actions to take.  
Even when several causes are possible, it is usually easiest to start from the closest symptom and work through the checks in order.

## `host.docker.internal` connection issues

- Symptom: the mediators (`mediator-owner` / `mediator-buyer`) or the frontend cannot reach `host.docker.internal` and DNS resolution fails
  - Example log: `ConnectError("dns error", ... "failed to lookup address information: nodename nor servname provided, or not known")`
  - Example (frontend, when `VITE_RPC_URL` of `iot-market-ui` is set to `host.docker.internal`): `JsonRpcProvider failed to detect network ... (perhaps the URL is wrong or the node is not started)` printed every second. If this appears with the default `http://127.0.0.1:8545`, the cause is not this one but a stopped node (check that the `npx hardhat node` terminal is still running)
  - Cause: `host.docker.internal` is a special hostname that Docker Desktop provides **only inside containers**. When you run a process **outside a container (natively on the host OS)** with `cargo run` or `npm run dev`, that name cannot be resolved.

### Check

- Confirm the node itself is alive by running this on the host:

  ```bash
  curl http://localhost:8545
  ```

  If it returns `{"jsonrpc":"2.0", ... "error": ... "Parse error: Unexpected end of JSON input"}`, the **node is reachable and healthy** — this is the normal JSON-RPC response to a GET with an empty body, not an error.

### Fix A — add a hosts entry (recommended)

As in [step 0.5 of Quickstart](../setup/quickstart.md), add `127.0.0.1 host.docker.internal` to your hosts file. Besides the settings files, `host.docker.internal` also appears in `mediator-owner/src/main.rs` (IPFS and PostgreSQL endpoints), `mediator-buyer/src/main.rs`, and the search API of `iot-market-ui`, so this is the most reliable fix.

### Fix A' — edit the settings and sources

If you cannot change the hosts file, replace `host.docker.internal` with `localhost` in the files listed above. The settings file looks like this.


- `mediator-owner/settings/owner_1.yaml`:

  ```yaml
  api_url: "http://localhost:3000"
  rpc_url: "http://localhost:8545"
  ```

- Apply the same change to the corresponding `mediator-buyer` settings and to the frontend (`iot-market-ui`) RPC endpoint configuration. If you cannot find where the endpoint is set, grep the repository for `host.docker.internal`.
- Restart each process after editing.

### Fix B — run inside containers

Run the mediators and frontend on the DevContainer / Docker side, where `host.docker.internal` resolves as intended. Choose this if you prefer not to change the settings.

## IPFS container keeps restarting

- Symptom: running `docker compose ps` in the `ipfs` directory shows the `ipfs` STATUS stuck at `Restarting`
  - Check: read the error with `docker compose logs ipfs`. If it says port 8080 is in use, check whether the publisher is running ([Port conflicts](#port-conflicts))
  - Action: if the stored data is broken, delete it together with the volumes with `docker compose down -v`, then recreate with `docker compose up -d`. The IPFS and PostgreSQL data are lost. Recreate the `ipfs_records` table as described in `ipfs/README.md`, then restart `mediator-owner` so it registers the data again

## `http://<HOST_IP>:5173/` not reachable from smartphone

- Symptom: the phone browser cannot open `http://<HOST_IP>:5173/` (`<HOST_IP>` is the PC's LAN IP)
  - Check: the frontend was started with `--host 0.0.0.0`
  - Check: both devices are on the same Wi-Fi network
  - Check: the firewall allows `5173`

## Product does not appear

- Symptom: no merchandise is shown at <http://localhost:5173>
  - The top page is a search screen. "Run Search" lists only items whose video metadata `mediator-owner` registered in IPFS and PostgreSQL (the `ipfs_records` table). Items created from event files and the 5 items registered by the deployment do not appear in the search. Open `http://localhost:5173/merchandise/<item address>` directly.
  - Check: `mediator-owner` was started with `cargo run -- settings/owner_1.yaml`
  - Check: the `mediator-owner` terminal shows `Watching raw_data/output for new event files`. If not, update the course repository (`git pull`) and restart it
  - Check: when an event file is written, `Product deployed: 0x... for file ...` is printed. The address shown is the one to use in the merchandise page URL

## Purchase does not succeed

- Symptom: purchase does not succeed
  - Check: MetaMask is connected with Account #2 (`0x3C44...`). A purchase requires the buyer's public key to be registered in the PubKey contract. `mediator-buyer` registers the key of Account #2 at startup, so a purchase from another account reverts
  - Action: after restarting the local chain, clear stale MetaMask state and reconnect

## Events are rejected with `no_matching_consent`

- Symptom: in the HA Demo Simulator (or similar), some datasets never appear in `/platform/ingest`. In `audit/logs` they show `action: deny` / `reason: no_matching_consent` / `subject_did: "unknown"`
  - Example log:

    ```json
    {"action":"deny","subject_did":"unknown","dataset_id":"home/event/possible_littering","purpose":"research","reason":"no_matching_consent", ...}
    ```

  - Likely causes: the consent was not registered, or **the consent has expired**. If the `valid_to` of a consent file (`examples/ha_demo/consent_*.json`) is in the past, the consent does not match at the current time and the event is denied even after registration. The bundled samples are valid until `2030-12-31T23:59:59Z`, but files whose dates you edited or files you wrote yourself can expire.

### Check

- Read the audit log first (in zsh, quote the URL because `?` is treated as a glob):

  ```bash
  curl 'http://localhost:8080/audit/logs?limit=20'
  ```

- Open the consent file for the denied `dataset_id` and confirm that **today's date** falls within `valid_from`–`valid_to`. If `valid_to` is in the past, the consent has expired.

### Fix

- Move `valid_to` to a later date and re-register. Example that rewrites `valid_to` in every file at once (macOS `sed` needs `-i ''`):

  ```bash
  sed -i '' 's/"valid_to": "[^"]*"/"valid_to": "2035-12-31T23:59:59Z"/' examples/ha_demo/consent_*.json
  ```

- After editing, re-register the consents (confirm each response contains `"status":"stored"`), re-send the events, then check `/platform/ingest` again.
- Existing `deny` entries remain in the log as history. Always verify with **a new event sent after the consent was registered**.

## Merchandise page returns 500 (iot-market-ui endpoint)

- Symptom: the list page (`/`) opens, but an individual merchandise page `http://localhost:5173/merchandise/<address>` returns **HTTP 500**
  - Cause: the frontend (`iot-market-ui`) reads its RPC and publisher endpoints from `iot-market-ui/.env.local`. If that endpoint **cannot reach the running Hardhat node** (someone else's IP, a wrong port such as `8546`, or a host that is not up), the merchandise page fails to fetch data and returns 500. The list page has a timeout and still renders, so the failure first surfaces on the detail page.

### Check and fix

- Open `iot-market-ui/.env.local` and confirm `VITE_RPC_URL` points at the running node (default `http://127.0.0.1:8545`). If the file is missing, copy the template.

  ```bash
  cd ~/program/Blockchain_IoT_Marketplace/iot-market-ui
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
  # Stop the running node (Terminal 1 of the Quickstart) with Ctrl+C first
  cd ~/program/Blockchain_IoT_Marketplace/iot-market
  npx hardhat node --hostname 0.0.0.0
  ```

  Restarting empties the chain. Redo step 2 (deploy) of the [Quickstart](../setup/quickstart.en.md) and restart `mediator-owner` and `mediator-buyer`. If MetaMask shows a nonce error, reset the account (Settings → Advanced → Clear activity tab data).

- Use the **PC's LAN IP** for the RPC URL in MetaMask (e.g. `http://192.168.1.20:8545`), with chain ID `31337`.
- Confirm the PC firewall allows `8545` and that the PC and phone are on the same Wi-Fi.
- If you use the frontend from the phone, set `iot-market-ui/.env.local` to the same LAN IP (see the previous item).

## `GET /platform/ingest` returns 404

- Symptom: `curl http://localhost:8080/platform/ingest` returns `{"detail":"platform_ingest_read_disabled"}` (404)
  - Cause: `GET /platform/ingest` is a demo-only API that returns every row without authentication, and it is disabled by default.
  - Fix: restart the publisher with `PLATFORM_INGEST_READ_ENABLED=true`.

    ```bash
    cd ~/program/Blockchain_IoT_Marketplace
    PLATFORM_INGEST_READ_ENABLED=true docker compose -f infra/docker-compose.yml up --build -d
    ```

## Port conflicts

- Symptom: a process does not start and prints `address already in use` or `port is already allocated`
  - Check: find the process that holds the port (macOS / Linux).

    ```bash
    lsof -i :8080
    ```

  - The main ports used in the course:

    | Port | Used by | Note |
    |---|---|---|
    | 8080 | IPFS gateway (`ipfs/docker-compose.yaml`) and the publisher (`infra/docker-compose.yml`) | Cannot run together. Before using the publisher, run `docker compose down` in the `ipfs` directory |
    | 8545 | Hardhat node (`npx hardhat node`) | Do not start a second node |
    | 5173 | Frontend (`iot-market-ui`, `npm run dev`) | Check that no other Vite dev server is running |
    | 8090 | assistant (`assistant` / `assistant-llm` / `assistant-demo` profiles) | The three profiles share the port, so start one at a time |
    | 18000 | llm-mock (`llm-mock` / `assistant-demo` profiles) | Stop the `llm-mock` / `assistant-demo` container before running the host `uvicorn examples.phase3_llm_mock_server:app --port 18000` |
    | 11434 | Ollama (installed directly on the PC) | If the Ollama app is running in the background, another Ollama cannot start |

  - Fix: stop the conflicting process and start again.
