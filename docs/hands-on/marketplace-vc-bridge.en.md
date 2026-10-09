# Hand a credential to the buyer's wallet (Bridge / Stage 5)

This is the core v2 hands-on. A marketplace purchase triggers the issuance of a PurchaseViewerVC to the buyer's phone, and presenting that VC lets the buyer fetch the data.

> **What you'll do**: Purchase → bridge → PurchaseViewerVC → wallet → ViewerToken → fetch
>
> **Prerequisites**: [Quickstart](../setup/quickstart.md) and [HA SSI Wallet](ha-ssi-wallet.md)
>
> **What you need**: PC + MetaMask + smartphone (iw3ip-wallet)
>
> **Time required**: approx. 60 min

!!! abstract "Hands-on for trying v2"
    Starting from a marketplace purchase, you issue a
    PurchaseViewerVC to the buyer's wallet and fetch data by
    presenting it, covering the whole flow.
    For design details, see the [Marketplace VC Bridge design spec](../design/marketplace-vc-bridge-spec.md).

!!! tip "Choosing a dataset"
    The example uses `home/env/temperature`, but the deploy script from
    Stage 6 case B onward also registers Merchandise #2
    (`home/event/possible_littering`) and Merchandise #4
    (`home/event/flood_risk_high`), so you can try the purchase flow with
    the same events as Stage 0
    (the Presentation Definition in
    [purchase-viewer-possible-littering.json](https://github.com/ertlnagoya/Blockchain_IoT_Marketplace/blob/main/examples/ssi_wallet/purchase-viewer-possible-littering.json)
    is already registered).

## Goal

In this hands-on, you check how authorization changes when **v2 (VC integration)**
is added to **v1 (the current marketplace + MetaMask + encrypted IPFS delivery)**.

- What it means for one person to handle **two identities** (an ETH key and a did:jwk)
- The **separation of roles** between ETH payment and VC authorization
  (payment and viewing rights sit in different layers)
- How the bridge service connects an on-chain Purchase event to
  off-chain VC issuance

## What this page covers

- v1 (decrypting `emitUpload(encryptURI)`) and v2 (via `PurchaseViewerVC`)
  **run side by side from the same purchase**
- `/marketplace/claim` is idempotent (posting the same tx_hash issues only one claim)
- The `eth_addr ↔ did:jwk` binding is recorded in the audit log
- `/platform/data?merchandise=<addr>` is gated by a ViewerToken

## Overview

```
[buyer]
   │  pick an item in iot-market-ui, purchase() with MetaMask
   ▼
[Merchandise.purchase tx]
   │  ↓                        ↓
   │  Purchase event          (optional) v1 lane: encryptURI delivery
   ▼
[bridge listener]     or      [iot-market-ui /purchased/[txHash]]
   │                           │
   └───────┬───────────────────┘
           │  POST /marketplace/claim (idempotent on tx_hash)
           ▼
       [publisher]
           │  issues a pre_authorized_code, returns a deeplink
           │  audit: marketplace/claim
           ▼
       [iw3ip-wallet]
           │  receives via OID4VCI from the deeplink
           │  → PurchaseViewerVC (claims: merchandise/tx/eth)
           │  audit: marketplace/issued (eth_did_bound)
           ▼
       [iw3ip-wallet]
           │  presents via OID4VP
           │  → ViewerToken (60s, multi-use)
           ▼
       [/platform/data?merchandise=<addr>]
           │  fetch with ViewerToken as Bearer
           │  audit: platform/data (viewer_token_used)
           ▼
       [JSON data retrieved]
```

## Prerequisites

- You have verified [Stage 1](ha-ssi-wallet.md) and [Stage 3](ha-ssi-viewer.md)
- You can start iot-market-ui (Svelte) and the Hardhat local chain
- iw3ip-wallet runs on a physical iPhone (connected to the Metro bundler)
- Check your LAN IP (`ipconfig getifaddr en0`). This page uses `192.168.68.53`,
  so replace it with the IP of your environment
- The steps assume a Mac and an iPhone, with the course repository cloned to `~/program/Blockchain_IoT_Marketplace` and the wallet cloned to `~/program/iw3ip-wallet`

---

## Step 1. Start the Hardhat node

### What to check

- The Hardhat local chain starts and 20 test accounts become available
- Note the private key of Account #2 for the buyer (you import it into MetaMask later)

### Steps

Terminal A (leave it running):

```bash
cd ~/program/Blockchain_IoT_Marketplace/iot-market
npx hardhat node --hostname 0.0.0.0
```

### Expected output (excerpt)

```
Started HTTP and WebSocket JSON-RPC server at http://0.0.0.0:8545/

Accounts
========
Account #0: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 (10000 ETH)
Private Key: 0xac0974bec...

Account #2: 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC (10000 ETH)
Private Key: 0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a
...
```

**Note down**: the private key of Account #2 (`0x5de4...`) and the address of Account #2 (`0x3C44...`).

---

## Step 2. Deploy the contracts

### What to check

- PubKey + IoTMarket + Merchandise×5 are deployed in one run
- Note the addresses of IoTMarket and the first Merchandise (the values are the same every time)

### Steps

Terminal B:

```bash
cd ~/program/Blockchain_IoT_Marketplace/iot-market
npx hardhat run scripts/deployMerchandiseWithIoTMarket.ts --network localhost
```

### Expected output

```
Contract "PubKey" with 0x5FbDB2315678afecb367f032d93F642f64180aa3 deployed
Contract "IoTMarket" with 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512 deployed
Contract "Merchandise" with 0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0 deployed
Merchandise 0 registered
Contract "Merchandise" with 0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9 deployed
Merchandise 1 registered
Contract "Merchandise" with 0x0165878A594ca255338adfa4d48449f69242Eb8F deployed
Merchandise 2 registered
Contract "Merchandise" with 0x2279B7A0a67DB372996a5FaB50D91eAA73d2eBe6 deployed
Merchandise 3 registered
Contract "Merchandise" with 0x610178dA211FEF7D417bC0e6FeD39F05609AD788 deployed
Merchandise 4 registered
```

**Note down**: IoTMarket = `0xe7f1725...` (you use this **many times** in this hands-on),
and the five Merchandise addresses (each purchase consumes one).

---

## Step 3. Register a PubKey (buyer preparation)

### What to check

- `Merchandise.purchase()` calls `i_pubKey.getPubKey(tx.origin)` internally, so
  the purchasing account (Account #2) must register a public key in the **PubKey** contract beforehand
- If the registration is missing, the purchase reverts with `PubKey__NotRegistered`

### Steps

Terminal B (Hardhat console):

```bash
cd ~/program/Blockchain_IoT_Marketplace/iot-market
npx hardhat console --network localhost
```

At the console prompt:

```javascript
const [marketOwner, iotOwner, buyer] = await ethers.getSigners();
const pubKey = await ethers.getContractAt(
  "PubKey",
  "0x5FbDB2315678afecb367f032d93F642f64180aa3",
  buyer
);
const tx = await pubKey.registerKey("[dummy-pubkey-for-handson]");
await tx.wait();
console.log("registered for:", await buyer.getAddress());
.exit
```

### Expected output

```
registered for: 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
```

**Format**: any string that starts with `[` and ends with `]` is accepted (`isPubKey()` checks the format only).
It is meant to be an RSA public key (used by the seller for encryption on the v1 path), but a dummy value
is enough here because this hands-on checks only the v2 path.

---

## Step 4. Start the publisher and the bridge

### What to check

- The publisher can issue the VC types including PurchaseViewerVC (if you have gone through Stage 7, all five types are available)
- The bridge subscribes to Hardhat Purchase events and watches Merchandise×5

### Steps

Terminal C:

```bash
cd ~/program/Blockchain_IoT_Marketplace
git checkout main && git pull --ff-only

# env for the bridge (LAN IP is the value from ipconfig getifaddr en0)
cat > infra/.env <<EOF
BRIDGE_HARDHAT_RPC=http://host.docker.internal:8545
BRIDGE_IOT_MARKET_ADDRESS=0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512
BRIDGE_PUBLIC_PUBLISHER_URL=http://192.168.68.53:8080
EOF

docker compose -f infra/docker-compose.yml \
  --profile ssi-wallet --profile mv-bridge \
  up --build -d
```

### Expected output (three checks)

#### 4-A. Publisher health check

```bash
sleep 5
curl -s http://192.168.68.53:8080/health
```

→ `{"status":"ok","service":"publisher"}`

#### 4-B. Is PurchaseViewerVC registered?

```bash
curl -s http://192.168.68.53:8080/.well-known/openid-credential-issuer | python3 -m json.tool | grep -A1 PurchaseViewerVC
```

→

```
"PurchaseViewerVC": {
  "format": "dc+sd-jwt",
  ...
"vct": "https://iw3ip.example/credentials/PurchaseViewerVC/v1",
```

#### 4-C. Has the bridge connected to Hardhat and discovered the Merchandise contracts?

```bash
docker logs iw3ip-mv-bridge 2>&1 | tail -5
```

→

```
bridge: listening to 5 merchandise(s)
bridge: starting poll from block 14
bridge: started rpc=http://host.docker.internal:8545 market=0xe7f1725... publisher=http://publisher:8080
```

!!! warning "Setting BRIDGE_PUBLIC_PUBLISHER_URL is required"
    Without it, the `credential_issuer` in the deeplink issued later in Step 6
    becomes the Docker-internal host name `http://publisher:8080`, and
    the iPhone wallet fails because it cannot fetch the metadata when it opens the deeplink.

---

## Step 5. Configure MetaMask

### What to check

- MetaMask can connect to Hardhat localhost (chain ID 31337)
- The private key of the buyer (Account #2) is imported and 10000 ETH is shown

### Steps

Use a browser that **supports the MetaMask extension, such as Chrome, Firefox, or Brave** (Safari does not work).

1. MetaMask extension icon → network name at the top → **"Add network"**
2. Enter:
    - Network Name: `Hardhat localhost`
    - RPC URL: `http://192.168.68.53:8545`
    - Chain ID: `31337`
    - Currency: `ETH`
3. After saving, switch to `Hardhat localhost`
4. Account menu → **"Import private key"** → the private key of Account #2 noted in Step 1

### Expected display

- Network: `Hardhat localhost`
- Address: `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC`
- Balance: `10000 ETH`

---

## Step 6. Purchase → the bridge captures the Purchase event

### What to check

- `Merchandise.purchase()` succeeds through MetaMask
- The bridge **detects the Purchase event by polling** and calls `/marketplace/claim`
- A new `marketplace/claim` row is left in the publisher's audit log

### Steps A: through the browser (the normal procedure)

Start iot-market-ui (Terminal D):

```bash
cd ~/program/Blockchain_IoT_Marketplace/iot-market-ui
cat > .env.local <<EOF
VITE_RPC_URL=http://192.168.68.53:8545
VITE_PUBLISHER_URL=http://192.168.68.53:8080
EOF
npm run dev -- --host 0.0.0.0 --port 5173
```

Open the following in a PC browser (Chrome, etc.):

```
http://192.168.68.53:5173/merchandise/0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9
```

(This is Merchandise #1. You do not have to use `#0`, but note that its state stays IN_PROGRESS.)

"Purchase" → confirm in MetaMask → the tx is sent. Once it is confirmed, the page moves automatically to
`/purchased/<txHash>?merchandise=...&dataset=...&buyer=...` and shows a QR code and a deeplink.

### Steps B: alternative procedure when MetaMask does not work

Terminal B (Hardhat console):

```bash
cd ~/program/Blockchain_IoT_Marketplace/iot-market
npx hardhat console --network localhost
```

```javascript
const [marketOwner, iotOwner, buyer] = await ethers.getSigners();
const merch = await ethers.getContractAt(
  "Merchandise",
  "0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9",
  buyer
);
const tx = await merch.purchase({ value: await merch.getPrice() });
const r = await tx.wait();
console.log("tx hash:", r.hash, "status:", r.status);
```

→ `tx hash: 0x4e7c...` `status: 1`

### Expected output (wait 5 seconds, then check)

#### 6-A. The bridge log shows the captured Purchase event

```bash
docker logs iw3ip-mv-bridge 2>&1 | tail -5
```

→

```
bridge: Purchase event from 0xDc64a140... buyer=0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC tx=0x4e7c3e0a...
bridge: claim ok jti=2b417b32e6566830 deeplink=openid-credential-offer://?credential_offer=...
```

Note down this `jti=...` value because you use it later.

#### 6-B. A marketplace/claim row in the audit log

```bash
curl -s 'http://192.168.68.53:8080/audit/logs?limit=2' | python3 -m json.tool
```

→

```json
{
  "raw_topic": "marketplace/claim",
  "subject_did": "eth:0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
  "purpose": "purchase",
  "reason": "claim_received:2b417b32e6566830:tx=0x4e7c3e0a...",
  ...
}
```

At this point the on-chain purchase has reached the publisher, and it is ready to issue the VC.

---

## Step 7. Receive the PurchaseViewerVC in the iPhone wallet

### What to check

- Open the bridge's deeplink in the iPhone wallet (iw3ip-wallet) and receive the issued VC
- The VC claims contain `merchandise_address` / `tx_hash` / `buyer_eth_addr`
- The publisher **records the eth_addr ↔ did:jwk binding in the audit log**

### Preparation: start the Metro bundler

If the wallet on the iPhone does not open and shows a `No script URL provided` error, Metro is not running.
Terminal E:

```bash
cd ~/program/iw3ip-wallet
npx react-native start
```

Confirm `Metro waiting on...`. Press "Reload JS" in the wallet to return to the normal screen.

### Steps: pass the deeplink to the iPhone as a QR code

```bash
DEEPLINK=$(docker logs iw3ip-mv-bridge 2>&1 | grep "claim ok jti=2b417b32e6566830" | tail -1 | sed -E 's/.*deeplink=//')
echo "$DEEPLINK"

# Open the QR code in the Mac browser
ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "$DEEPLINK")
open "https://api.qrserver.com/v1/create-qr-code/?size=400x400&data=$ENCODED"
```

Scan the QR code with the iPhone camera. The wallet starts and shows the approval screen for "IW3IP Purchase Viewer Credential", so approve it.

!!! warning "QR generation uses an external service"
    The command above sends the content of the deeplink to an external QR generation service (api.qrserver.com). The deeplink contains the code used for VC issuance, so do not use this outside a local hands-on environment.

### Expected result

#### 7-A. Expanding the claims in the wallet's VC list shows the following

- `dataset_id`: `home/env/temperature`
- `merchandise_address`: `0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9`
- `buyer_eth_addr`: `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC`
- `tx_hash`: `0x4e7c3e0a...`
- `allowed_actions`: `["read"]`

#### 7-B. The eth↔did binding is recorded in the audit log

```bash
curl -s 'http://192.168.68.53:8080/audit/logs?limit=3' | python3 -m json.tool
```

→

```json
{
  "raw_topic": "marketplace/issued",
  "subject_did": "did:jwk:eyJhbG...",
  "purpose": "purchase_link",
  "reason": "eth_did_bound:claim=2b417b32...:eth=0x3C44...:tx=0x4e7c3e0a...",
  "holder_did": "did:jwk:eyJhbG..."
}
```

**Once you see this `eth_did_bound` row, the main check of Stage 5 is complete.**
The MetaMask key and the wallet key are now tied together on the publisher as "the same person".

---

## Step 8. Obtain a ViewerToken

### What to check

- **Presenting the PurchaseViewerVC** in the wallet yields a ViewerToken (60 seconds, multi-use)
- Select the purchase-linked VC, as distinct from ConsentVC (Stage 1) and ViewerVC (Stage 3)

### Steps

In a PC browser:

```
http://192.168.68.53:8080/verifier/request?dataset_id=home/env/temperature&vc_kind=PurchaseViewerVC
```

`vc_kind=PurchaseViewerVC` is required (without it, the Presentation Definition for ConsentVC is selected).

Open the iPhone wallet from the QR code or the deeplink, then **select the PurchaseViewerVC and present it**.

Extract the token from the publisher log.

```bash
PUB=$(docker ps -qf name=publisher)
TOKEN=$(docker logs $PUB 2>&1 \
  | grep "viewer_token_issued vc_kind=PurchaseViewerVC" | tail -1 \
  | sed -E 's/.*token=([^ ]+).*/\1/')
echo "TOKEN=$TOKEN"
```

### Expected output

```
TOKEN=oJVsNtb5Un1NginSyyCcJavThu9WkTxRT6b8uhmWjRc
```

(The full log line is also emitted: `viewer_token_issued vc_kind=PurchaseViewerVC jti=cc38f06e... token=... dataset=home/env/temperature ttl=60s`)

---

## Step 9. Fetch the data (reverse lookup with `merchandise=<addr>`)

### What to check

- You can call `/platform/data` with the ViewerToken
- When you pass `merchandise=<contract address>` in the query, the publisher **reverse-looks up the dataset_id** and returns the same result
- After expiry (60 seconds), the call returns 401

### Steps (within 60 seconds)

```bash
MERCHANDISE=0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9

curl -s -H "Authorization: Bearer $TOKEN" \
  "http://192.168.68.53:8080/platform/data?merchandise=$MERCHANDISE" | python3 -m json.tool
```

### Expected output

```json
{
  "dataset_id": "home/env/temperature",
  "count": 0,
  "read_count": 1,
  "rows": []
}
```

`count: 0` is not a problem, because authorization has passed. You can load actual data separately with
`POST /platform/ingest` (Stage 1), but the purpose of this hands-on is to check the authorization logic.

### Checking expiry (after 60 seconds)

```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "http://192.168.68.53:8080/platform/data?dataset_id=home/env/temperature" | python3 -m json.tool
```

→

```json
{
  "detail": "viewer_token_expired"
}
```

This is also normal behavior, as specified in Stage 5 / Stage 3.

---

## Step 10. The audit log as a whole

### What to check

- One purchase leaves a chain of **at least four rows** in the audit log
- You can read the correspondence between on-chain (ETH key) and off-chain (did:jwk)

### Steps

```bash
curl -s 'http://192.168.68.53:8080/audit/logs?limit=10' | python3 -m json.tool
```

### Expected output (newest first)

| id | raw_topic | reason | subject |
| --- | --- | --- | --- |
| (latest) | `platform/data` | `viewer_token_used:cc38f06e...:1` | `did:jwk:...` |
| -1 | `oid4vp/response` | `ok` | `did:jwk:...` |
| -2 | `marketplace/issued` | **`eth_did_bound:claim=2b417b32...:eth=0x3C44...:tx=0x4e7c...`** | `did:jwk:...` |
| -3 | `marketplace/claim` | `claim_received:2b417b32...:tx=0x4e7c...` | `eth:0x3C44...` |

Confirm that `marketplace/claim` (subject: eth_addr) and `marketplace/issued` (subject: did:jwk) are **linked by the same claim_id**. This is the record that ties the ETH address that paid to the DID that presented the VC.

---

## Comparison with the v1 lane

With the same purchase, the v1 path (encryptURI delivery; called the v1 lane below) also runs in parallel.
If the seller calls `Merchandise.emitUpload(encryptURI)`,
the buyer's MetaMask side still sees the encrypted URI as before.

| Aspect | v1 (`emitUpload`) | v2 (`PurchaseViewerVC`) |
| --- | --- | --- |
| Delivery form | encryptURI (public-key encryption) | publisher API (Bearer) |
| Key management | ETH key in MetaMask | did:jwk in the wallet |
| Data source | IPFS (or anything else) | the publisher's `/platform/data` |
| Audit log | on-chain Upload event only | publisher audit log (including eth_did_bound) |
| Revocation | re-encrypt on key leakage | discard the VC in the wallet |

---

## Completion matrix

Stage 5 is complete when you have confirmed every item in the table below.

| Step | Check item | Example output / location |
| --- | --- | --- |
| Step 1 | Hardhat node started | `Started HTTP and WebSocket JSON-RPC server at http://0.0.0.0:8545/` |
| Step 2 | Contracts deployed | `Contract "IoTMarket" with 0xe7f1725...` |
| Step 3 | PubKey registered | `registered for: 0x3C44...` |
| Step 4-A | Publisher started | `{"status":"ok","service":"publisher"}` |
| Step 4-B | PurchaseViewerVC exposed | `"vct": ".../PurchaseViewerVC/v1"` in the issuer metadata |
| Step 4-C | Bridge connected to Hardhat | `bridge: listening to 5 merchandise(s)` |
| Step 5 | MetaMask balance | `10000 ETH` (Account #2) |
| Step 6-A | Bridge detected the Purchase | `bridge: claim ok jti=...` |
| Step 6-B | marketplace/claim in the audit log | `subject=eth:0x3C44...`, `reason=claim_received:...` |
| Step 7-A | VC received in the wallet | `merchandise_address` / `tx_hash` / `buyer_eth_addr` in the claims |
| Step 7-B | eth_did_bound in the audit log | `raw_topic=marketplace/issued`, `holder_did=did:jwk:...` |
| Step 8 | ViewerToken issued | publisher log `viewer_token_issued vc_kind=PurchaseViewerVC` |
| Step 9 | Fetch by merchandise reverse lookup | `{"dataset_id": "home/env/temperature", "read_count": 1}` |
| Step 9 (cont.) | 401 after 60 seconds | `{"detail": "viewer_token_expired"}` |
| Step 10 | Chain of four audit log rows | `marketplace/claim` → `marketplace/issued` → `oid4vp/response` → `platform/data` |

---

## Troubleshooting

These are the problems that occurred during verification on real devices, and how to handle them.

### A. MetaMask cannot send the tx because of a `chainId error`

```
MetaMask - RPC Error: Trying to send a raw transaction with an invalid chainId.
```

**Cause**: After the Hardhat node was restarted, an old nonce / chainId cache remains in MetaMask.

**Fix**: MetaMask → Settings → Advanced → **"Clear activity tab data"** or **"Reset Account"**.
If that does not solve it, you can make the purchase with **Step 6 Steps B (Hardhat console)** instead. That is enough to check the behavior of the bridge and the publisher.

### B. Purchase reverts with `0x295f0a57`

```
Error: VM Exception while processing transaction: reverted with an unrecognized custom error
```

**Cause**: `PubKey__NotRegistered`. You skipped Step 3, or registered the PubKey with an account other than Account #2.

**Fix**: Run Step 3 again. In the Hardhat console, call `pubKey.registerKey("[...]")` as the **buyer (Account #2)**.

### C. The iPhone wallet crashes as soon as it opens the deeplink / shows nothing

**Cause 1**: The `credential_issuer` in the deeplink issued by the bridge is
`http://publisher:8080` (the Docker-internal host name), which the iPhone cannot reach.

**Fix**: Set `BRIDGE_PUBLIC_PUBLISHER_URL=http://<LAN_IP>:8080` in `infra/.env` and
restart the bridge (Step 4). Fetch the deeplink again and confirm that the `credential_issuer` inside it
is the LAN IP:

```bash
DEEPLINK=$(docker logs iw3ip-mv-bridge 2>&1 | grep "claim ok" | tail -1 | sed -E 's/.*deeplink=//')
echo "$DEEPLINK" | python3 -c "import sys,urllib.parse,json; d=urllib.parse.unquote(sys.stdin.read().split('credential_offer=',1)[1]); print(json.loads(d)['credential_issuer'])"
# → http://192.168.68.53:8080  (not publisher:8080)
```

**Cause 2**: The wallet shows the `No script URL provided` error screen, which means the Metro bundler is not running.

**Fix**: Run `cd ~/program/iw3ip-wallet && npx react-native start` in another terminal and
press "Reload JS" in the wallet.

**Cause 3**: The wallet still holds an old cache of the issuer metadata.

**Fix**: **Fully quit** the wallet from the iPhone app switcher → start it again.

### D. The bridge log never shows `Purchase event`

```
bridge: listening to 5 merchandise(s)
@TODO TypeError: results is not iterable
```

**Cause**: Old code (filter subscription) remains. The latest code has been changed to `getLogs` polling.

**Fix**:

```bash
cd ~/program/Blockchain_IoT_Marketplace
git checkout main && git pull --ff-only
docker compose -f infra/docker-compose.yml --profile mv-bridge up --build -d bridge
```

### E. The iPhone cannot reach `192.168.68.53:8080/health`

**Cause**: The PC and the phone are on different networks, or the Mac's LAN IP has changed.

**Fix**:

```bash
ipconfig getifaddr en0   # current LAN IP
```

If it has changed, update `BRIDGE_PUBLIC_PUBLISHER_URL` in `infra/.env` and restart the bridge.

---

## Limits of this hands-on (future work)

- **Protection against impersonation of eth_addr ↔ did:jwk**: the current minimal implementation simply
  trusts the POST from the bridge / front end. In production, the holder must prove
  "this tx is mine" with an EIP-712 signature (for details, see [design spec §7.2](../design/marketplace-vc-bridge-spec.md))
- **Discovering the dataset_id**: currently hardcoded in the query string by the front end.
  The extension that reads it dynamically from the Merchandise `additionalInfo` is covered in [Stage 6](marketplace-vc-end-to-end.md)
- **Integration with ServiceVC**: the mechanism that authorizes continuous MQTT writes with a VC (Stage 4 prep) and
  the mechanism that reads data in connection with a purchase (Stage 5) are independent. A scenario that combines both is
  covered in [Stage 6](marketplace-vc-end-to-end.md)

## Related

- [Marketplace VC Bridge design spec (v1/v2)](../design/marketplace-vc-bridge-spec.md) — overall structure
- [SSI Wallet (Stage 1)](ha-ssi-wallet.md) — single-use write authorization
- [SSI Viewer (Stage 3)](ha-ssi-viewer.md) — multi-use read authorization (v1, dataset specified directly)
- [SSI Service (Stage 4 prep)](ha-ssi-service.md) — M2M continuous writes
