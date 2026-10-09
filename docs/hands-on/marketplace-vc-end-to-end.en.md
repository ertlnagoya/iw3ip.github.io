# Tie four credentials together end-to-end (Stage 6)

This is the capstone exercise for Stages 1–5. In one session, you run a scenario in which a Seller writes data with a ServiceVC and a Buyer reads it with a PurchaseViewerVC.

> **What you'll do**: Tie four VCs (Consent / Service / Viewer / PurchaseViewer) together in one flow
>
> **Prerequisites**: [Marketplace VC Bridge](marketplace-vc-bridge.md) (Stage 5)
>
> **What you need**: PC + MetaMask + smartphone (iw3ip-wallet)
>
> **Time required**: approx. 90 min

!!! abstract "Capstone exercise for Stages 1–5"
    In one session, you run a scenario in which the Seller writes data
    continuously with a **ServiceVC** and the Buyer reads it with a
    **PurchaseViewerVC**. The backend is the one from Stages 1–5 plus a change that
    obtains the dataset_id dynamically from the Merchandise `additionalInfo` (this site calls it Stage 6 case B).

!!! tip "Choosing a dataset"
    The example uses `home/env/temperature`, but the deploy script (Stage 6 case B)
    also creates Merchandise contracts for `home/event/possible_littering` and
    `home/event/flood_risk_high` at the same time, so you can also run
    Service ingest → Purchase → Viewer fetch with the same event format as Stage 0
    (the ServiceVC / PurchaseViewerVC PDs for each dataset are already registered).

## Goal

- Confirm that the four VC types involved in the data flow (ConsentVC / ViewerVC /
  **ServiceVC** / **PurchaseViewerVC**) **cooperate through one dataset**
  (the fifth type, SellerVC, handles the seller's identity and
  is covered in [Stage 7](marketplace-seller-vc.md))
- Understand the **division of roles** between on-chain payment (MetaMask) and
  off-chain authorization (VC) through one continuous flow
- Read the links in the audit log: follow the relationship ETH key → did:jwk →
  ServiceVC holder → PurchaseViewerVC holder

## What this page covers

- Data written with a ServiceVC **can be read correctly with a PurchaseViewerVC
  for the same dataset** (Stage 4 prep joined with Stage 5)
- With Stage 6 case B, the Merchandise **holds the dataset_id on-chain**, and
  the bridge / iot-market-ui resolve the dataset without hardcoding
- The links in the audit log that remain for one dataset

## Overview

```
[Seller]                              [Buyer]
   │                                     │
   │ present ServiceVC in the wallet     │
   │  → ServiceToken (1h)                │
   │                                     │
   │ POST /platform/ingest x N           │
   │  → accumulates in app.state.ingested│
   │                                     │
   │                                     │ Merchandise.purchase() with MetaMask
   │                                     │  → Purchase event
   │                                     ▼
   │                                  [bridge listener]
   │                                     │ resolves dataset_id from
   │                                     │ Merchandise.additionalInfo
   │                                     │ POST /marketplace/claim
   │                                     ▼
   │                                  [publisher]
   │                                     │ PurchaseViewerVC offer
   │                                     ▼
   │                                  [iw3ip-wallet]
   │                                     │ receive via OID4VCI → eth↔did binding
   │                                     │ present via OID4VP → ViewerToken
   │                                     ▼
   │                                  GET /platform/data?merchandise=<addr>
   │                                     │
   │  ── the N rows written by the Seller reach the Buyer ──
```

## Prerequisites

- You have gone through [Stage 1](ha-ssi-wallet.md) / [Stage 3](ha-ssi-viewer.md) /
  [Stage 4 prep](ha-ssi-service.md) / [Stage 5](marketplace-vc-bridge.md)
- The existing publisher + bridge + Hardhat + iot-market-ui are assumed to stay running
  and to be used as they are
- This page uses `192.168.68.53` as the LAN IP, so replace it with the IP of your environment
- The steps assume a Mac and an iPhone, with the course repository cloned to `~/program/Blockchain_IoT_Marketplace` and the wallet cloned to `~/program/iw3ip-wallet`
- In this hands-on, **one iPhone wallet plays two roles (seller + buyer)**
  for a single person (separating seller and buyer as in real operation is future work)

---

## Step E0. Start the prerequisite environment (same as Stage 5)

```bash
# Hardhat + deploy (latest version, in which Merchandise holds dataset_id in additionalInfo)
cd ~/program/Blockchain_IoT_Marketplace/iot-market
git checkout main && git pull --ff-only
npx hardhat node --hostname 0.0.0.0 &   # Terminal A recommended
npx hardhat run scripts/deployMerchandiseWithIoTMarket.ts --network localhost
# Note IoTMarket = 0xe7f1725... and the addresses of Merchandise #0..4

# Register a PubKey (buyer = Account #2)
npx hardhat console --network localhost
# > const [_, __, buyer] = await ethers.getSigners();
# > const pk = await ethers.getContractAt("PubKey", "0x5FbDB23...", buyer);
# > await (await pk.registerKey("[handson]")).wait();
# > .exit

# publisher + bridge
cd ~/program/Blockchain_IoT_Marketplace
docker compose -f infra/docker-compose.yml --profile ssi-wallet --profile mv-bridge up --build -d

# Check the bridge log
docker logs iw3ip-mv-bridge 2>&1 | tail -5
# → bridge: listening to 5 merchandise(s)

# iot-market-ui (another terminal)
cd iot-market-ui
npm run dev -- --host 0.0.0.0 --port 5173
```

If you get stuck, see [Troubleshooting in the Stage 5 hands-on](marketplace-vc-bridge.md#troubleshooting).

---

## Step E1. Seller phase — continuous writes with a ServiceVC

### What to check

- Presenting one ServiceVC is enough for **multiple** ingests to pass
- The dataset written to (`home/env/temperature`) is what the buyer reads later

### Steps

In a PC browser (Mac):

```
http://192.168.68.53:8080/issuer/offer?type=ServiceVC&dataset_id=home/env/temperature&purpose=write_continuous
```

Show the QR code → scan it with the iPhone wallet → **approve "IW3IP Service Credential"**.

Then present it:

```
http://192.168.68.53:8080/verifier/request?dataset_id=home/env/temperature&vc_kind=ServiceVC
```

QR code → **select the ServiceVC** in the wallet and present it.

### Extract the token → five consecutive ingests

```bash
PUB=$(docker ps -qf name=publisher)
SERVICE=$(docker logs $PUB 2>&1 | grep "service_token_issued" | tail -1 | sed -E 's/.*token=([^ ]+).*/\1/')

for i in 1 2 3 4 5; do
  curl -s -X POST http://192.168.68.53:8080/platform/ingest \
    -H "Authorization: Bearer $SERVICE" \
    -H "Content-Type: application/json" \
    -d "{\"dataset_id\":\"home/env/temperature\",\"value\":$((30 + i))}" \
    | python3 -c "import json,sys;d=json.load(sys.stdin);print('iter',$i,d)"
done
```

### Expected output

```
iter 1 {'status': 'received', 'count': N}
iter 2 {'status': 'received', 'count': N+1}
... (increases sequentially up to N+4)
```

Five rows with **values 31–35** have now been accumulated in `home/env/temperature` in the seller role.

---

## Step E2. Buyer phase — purchase a Merchandise

### What to check

- With Stage 6 case B, the bridge **obtains** the `dataset_id` written in the Merchandise
  `additionalInfo` **automatically from on-chain**
- The five Merchandise contracts have different datasets (temperature / littering / flood). In this hands-on,
  choose a temperature Merchandise

### Datasets of the Merchandise contracts

| Merchandise | Address | dataset_id |
|---|---|---|
| #0 | `0x9fE46736679d2D9a65F0992F2272dE9f3c7fa6e0` | `home/env/temperature` |
| #1 | `0xDc64a140Aa3E981100a9becA4E685f962f0cF6C9` | `home/env/temperature` |
| #2 | `0x0165878A594ca255338adfa4d48449f69242Eb8F` | `home/event/possible_littering` |
| #3 | `0x2279B7A0a67DB372996a5FaB50D91eAA73d2eBe6` | `home/env/temperature` |
| #4 | `0x610178dA211FEF7D417bC0e6FeD39F05609AD788` | `home/event/flood_risk_high` |

(The contracts are deployed deterministically, so the values are the same every time.)

You want to read the temperature data written with the ServiceVC, so choose a **temperature Merchandise** that has not been purchased yet (for example, `#3`).

### Steps (using the Hardhat console)

```bash
cd ~/program/Blockchain_IoT_Marketplace/iot-market
npx hardhat console --network localhost
```

```javascript
const [_, __, buyer] = await ethers.getSigners();
const merch = await ethers.getContractAt(
  "Merchandise",
  "0x2279B7A0a67DB372996a5FaB50D91eAA73d2eBe6",   // Merchandise #3
  buyer
);
const tx = await merch.purchase({ value: await merch.getPrice() });
const r = await tx.wait();
console.log("tx hash:", r.hash, "status:", r.status);
```

→ `tx hash: 0x...`, `status: 1`

### Expected: the bridge resolves the dataset_id from on-chain

```bash
docker logs iw3ip-mv-bridge 2>&1 | grep "Purchase event" | tail -1
```

→ The log contains **`dataset=home/env/temperature`**:

```
bridge: Purchase event from 0x2279B7A0... buyer=0x3C44... dataset=home/env/temperature tx=0x...
bridge: claim ok jti=...
```

Confirm that the dataset is obtained from on-chain and is not a hardcoded value.

```bash
docker logs iw3ip-mv-bridge 2>&1 | grep "claim ok" | tail -1
# → extract the jti and the deeplink
```

---

## Step E3. Buyer phase — receive the PurchaseViewerVC

### What to check

- The receiving procedure is the same as in Stage 5, but the target dataset **matches the one the seller wrote to**

```bash
DEEPLINK=$(docker logs iw3ip-mv-bridge 2>&1 | grep "claim ok" | tail -1 | sed -E 's/.*deeplink=//')
ENCODED=$(python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1]))" "$DEEPLINK")
open "https://api.qrserver.com/v1/create-qr-code/?size=400x400&data=$ENCODED"
```

Scan the QR code with the iPhone wallet and approve "IW3IP Purchase Viewer Credential". The command above sends the deeplink to an external QR generation service, so do not use it outside a local hands-on environment.

Confirm in the wallet that the VC claims contain `dataset_id: home/env/temperature`.

audit:

```bash
curl -s 'http://192.168.68.53:8080/audit/logs?limit=3' | python3 -m json.tool | grep -A1 marketplace/issued
```

→ `eth_did_bound:claim=...:eth=0x3C44...:tx=0x...`

---

## Step E4. Buyer phase — obtain a ViewerToken and fetch the data

### What to check

- Presenting the PurchaseViewerVC yields a ViewerToken (Stage 5)
- Fetching with `merchandise=<addr>` goes through the dataset_id resolved by the bridge, and
  **you can read the values 31–35 that the seller wrote in Step E1**
- This is the point Stage 6 is meant to confirm

### Steps

Request a presentation in a PC browser:

```
http://192.168.68.53:8080/verifier/request?dataset_id=home/env/temperature&vc_kind=PurchaseViewerVC
```

QR code → in the wallet, select the **PurchaseViewerVC** (do not confuse it with the ServiceVC or the ViewerVC) and present it.

```bash
PUB=$(docker ps -qf name=publisher)   # needed when running in a different terminal from Step E1
TOKEN=$(docker logs $PUB 2>&1 | grep "viewer_token_issued vc_kind=PurchaseViewerVC" | tail -1 | sed -E 's/.*token=([^ ]+).*/\1/')
echo "TOKEN=$TOKEN"

MERCHANDISE=0x2279B7A0a67DB372996a5FaB50D91eAA73d2eBe6
curl -s -H "Authorization: Bearer $TOKEN" \
  "http://192.168.68.53:8080/platform/data?merchandise=$MERCHANDISE" | python3 -m json.tool
```

### Expected output

```json
{
  "dataset_id": "home/env/temperature",
  "count": 5,
  "read_count": 1,
  "rows": [
    {"dataset_id": "home/env/temperature", "value": 31},
    {"dataset_id": "home/env/temperature", "value": 32},
    {"dataset_id": "home/env/temperature", "value": 33},
    {"dataset_id": "home/env/temperature", "value": 34},
    {"dataset_id": "home/env/temperature", "value": 35}
  ]
}
```

`count` is 5 only when nothing else has been written to this dataset. If you ingested into the same dataset before Step E1, the count is higher by that amount.

The Buyer has read the five rows that the Seller wrote in Step E1. This completes Stage 6.

---

## Step E5. The audit log chain

### What to check

- For one dataset, multiple subjects and operations are recorded in order

```bash
curl -s 'http://192.168.68.53:8080/audit/logs?limit=20' | python3 -m json.tool \
  | grep -E "raw_topic|reason|holder_did|subject_did" | head -40
```

Expected chain (newest first, excerpt):

| Stage | raw_topic | reason | Subject |
| --- | --- | --- | --- |
| 1. Buyer reads | `platform/data` | `viewer_token_used:<jti>:1` | did:jwk:... (buyer) |
| 2. Buyer presents | `oid4vp/response` | `ok` | did:jwk:... (buyer) |
| 3. Buyer receives | `marketplace/issued` | `eth_did_bound:claim=...:eth=0x3C44...:tx=...` | did:jwk:... (buyer) |
| 4. Bridge hand-off | `marketplace/claim` | `claim_received:<id>:tx=...` | eth:0x3C44... |
| 5. Seller writes x5 | `platform/ingest` | `service_token_used:<jti>:1–5` | did:jwk:... (seller) |
| 6. Seller presents | `oid4vp/response` | `ok` | did:jwk:... (seller) |

**Note**: In this hands-on the same wallet acts as both seller and buyer, so
the holder_did is identical. In real operation they are separate devices and separate did:jwk values.

### Points to observe

- What links (4) and (5) is the **dataset_id**: both are `home/env/temperature`
- (4) has an ETH subject → it is bound to a did:jwk in (3) → the read in (1) is done as that did:jwk,
  so **three forms of identity** are chained
- The ServiceVC's `service_token_used` occurs five times in a row → **it is read once with `viewer_token_used:1`, and the five rows arrive together**

---

## Completion matrix

| Step | Check item | Example output |
| --- | --- | --- |
| E0 | Environment started | `bridge: listening to 5 merchandise(s)` |
| E1 | ServiceVC received | "IW3IP Service Credential" in the wallet |
| E1 | ServiceToken issued | publisher log `service_token_issued ttl=3600s` |
| E1 | Five consecutive ingests | iter 1–5 all `status: received` |
| E2 | Bridge resolves the dataset_id on-chain | `dataset=home/env/temperature` |
| E2 | Bridge claim issued | `claim ok jti=...` |
| E3 | PurchaseViewerVC received | "IW3IP Purchase Viewer Credential" in the wallet |
| E3 | eth_did_bound | audit `marketplace/issued` |
| E4 | Five rows fetched by reverse lookup from the merchandise | `count: 5, rows: [..., {value: 31}, ..., {value: 35}]` |
| E5 | The audit chain can be read | `service_token_used` and `viewer_token_used` are chained on the same dataset |

---

## Comparison with the v1 lane (recap)

| Aspect | v1 (`emitUpload`) | v2 / Stage 6 |
| --- | --- | --- |
| Seller identity | implicit (owner of the Merchandise) | holder_did of the ServiceVC |
| Seller writes | (uploaded to IPFS off-line) | ingest to the publisher (left in the audit log) |
| Buyer identity | MetaMask eth address only | eth + did:jwk (bound by eth_did_bound) |
| Data delivery | encryptURI decryption | publisher API (Bearer ViewerToken) |
| Audit coverage | on-chain Upload event only | 10 rows in the publisher audit log (seller presentation 1 + writes 5 + bridge claim 1 + VC issuance 1 + buyer presentation 1 + read 1) |

---

## Troubleshooting

### A. In Step E2, a fallback value appears instead of `dataset=home/env/temperature`

**Cause**: The Merchandise is from an old deployment and has no `dataset_id` (a deployment from before Stage 6 case B).

**Fix**: Restart the Hardhat node (`Ctrl+C` → `npx hardhat node --hostname 0.0.0.0`) and then
redeploy with the latest `deployMerchandiseWithIoTMarket.ts`. MetaMask
needs its chainId cache reset ([Troubleshooting A in Stage 5](marketplace-vc-bridge.md#troubleshooting)).

### A2. `SERVICE` is empty at the start of Step E1

```
SERVICE=
```

**Cause**: You have just restarted the publisher in Step E0, so the past
`service_token_issued` logs are gone, and you have not yet completed the issuance and
presentation of Step E1.

**Fix**: Run the two steps "**issue** the ServiceVC → receive it in the wallet →
**present** the ServiceVC", and then grep the token. Issuance and
presentation use different URLs (`/issuer/offer` and `/verifier/request`), so
run both.

### B. `count: 0` in Step E4

**Cause**: `app.state.ingested` has been cleared, for example because the ServiceToken ingest in Step E1
ran in a different publisher session from the presentation (the publisher was restarted).

**Fix**: Do not restart the publisher between Step E1 and Step E4.
If you have restarted it, start over from Step E1.

### C. Other issues

See also [Troubleshooting in Stage 5](marketplace-vc-bridge.md#troubleshooting).

---

## Limits

- **Seller and buyer share one wallet**: true seller-buyer separation needs two
  iPhones or a UI that supports switching accounts inside the wallet
- **SellerVC**: the mechanism that requires an identity VC when a seller registers a
  `Merchandise` is not covered on this page (it is covered in [Stage 7](marketplace-seller-vc.md))
- **Observing continuity**: load testing of the case where multiple buyers read within the
  1-hour TTL of the ServiceVC (multi-tenant read) has not been done
- **EIP-712 signatures**: protection against impersonation of eth_addr ↔ did:jwk is still at the minimal-implementation stage

## Related

- [Marketplace VC Bridge design spec](../design/marketplace-vc-bridge-spec.md)
- [SSI Wallet (Stage 1)](ha-ssi-wallet.md)
- [SSI Viewer (Stage 3)](ha-ssi-viewer.md)
- [SSI Service (Stage 4 prep)](ha-ssi-service.md)
- [Marketplace × Wallet bridge (Stage 5)](marketplace-vc-bridge.md)
