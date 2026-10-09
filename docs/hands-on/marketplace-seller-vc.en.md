# Back the seller's identity (Seller VC, Stage 7)

This page adds a layer that uses a VC to back "who" is selling a dataset. It covers the fifth VC type, SellerVC.

> **What you'll do**: Issue and present a SellerVC to back the seller's identity
>
> **Prerequisites**: [Marketplace VC end-to-end](marketplace-vc-end-to-end.md) (Stage 6)
>
> **What you need**: PC + MetaMask + smartphone (iw3ip-wallet)
>
> **Time required**: approx. 45 min

!!! abstract "The fifth VC type: SellerVC"
    This adds a layer that uses a VC to back **"who"** is selling
    a dataset when it is listed on the marketplace. As in Stages 1–6, you proceed in the order
    present → obtain a token → call the API. For design details, see the [SellerVC design spec](../design/seller-vc-spec.md).

!!! tip "Choosing licensed_datasets"
    The example uses `home/env/temperature,home/env/humidity`, but for
    consistency with Stage 0 you can also use `home/event/possible_littering,home/event/flood_risk_high`
    (already registered in the publisher's `DEFAULT_ALLOWED_PURPOSES`).
    The `licensed_datasets` of a SellerVC is an array, so you can also include both.

## Goal

- Handle the **fifth VC** (SellerVC), which follows
  ConsentVC / ViewerVC / ServiceVC / PurchaseViewerVC
- Right after listing an item with `IoTMarket.registerMerchandise()`, the Seller
  calls **`/marketplace/register`** on the publisher and
  proves with a VC that "I am selling this Merchandise"
- Confirm the flow in which **`seller_did`** is returned when the Buyer
  reads `/platform/data?merchandise=<addr>`

## What this page covers

- How the `licensed_datasets` of a SellerVC states in the VC
  "which datasets this seller may sell"
- The SellerToken (24h, multi-use) works only for `/marketplace/register`
- (Optional) When the environment variable `MARKETPLACE_HARDHAT_RPC` is set, the publisher
  verifies on-chain that `Merchandise.getOwner() == seller_eth_addr`
- A **`marketplace/seller_registered`** row is added to the audit log
- The response on the buyer side contains **`seller_did`**, so the buyer can check who sold the data

## Overview

```
[Seller wallet]
   │ 1. /issuer/offer?type=SellerVC → receive
   │ 2. /verifier/request?vc_kind=SellerVC → present
   │ 3. SellerToken (24h, multi-use)
   │
[Seller (Hardhat console)]
   │ 4. deploy a Merchandise + IoTMarket.registerMerchandise()
   │
[Seller (curl or /seller UI)]
   │ 5. POST /marketplace/register
   │    Authorization: Bearer SellerToken
   │    body: merchandise_address + seller_eth_addr + tx_hash + dataset_id
   │
[publisher]
   │ - verifies the SellerToken (licensed_datasets contains dataset_id)
   │ - (optional) verifies Merchandise.getOwner() on-chain
   │ - audit: raw_topic=marketplace/seller_registered
   │ - merchandise → seller_did binding inside the publisher
   │
[Buyer (purchase flow of Stage 5/6)]
   │ → GET /platform/data?merchandise=<addr>
   │   the response carries seller_did
```

## Prerequisites

- You have gone through the Stage 5 / 6 hands-on
- The deploy script is the version that creates Merchandise contracts with `dataset_id` in `additionalInfo` (Stage 6 or later)
- The publisher is running with code that supports Stage 7 (the latest `main`)
- iw3ip-wallet (iPhone) works
- This page uses `192.168.68.53` as the LAN IP, so replace it with the IP of your environment
- The steps assume a Mac and an iPhone, with the course repository cloned to `~/program/Blockchain_IoT_Marketplace` and the wallet cloned to `~/program/iw3ip-wallet`

---

## Step S0. Bring the environment up to the latest code

```bash
cd ~/program/Blockchain_IoT_Marketplace
git checkout main && git pull --ff-only

# (Optional) only when enabling on-chain owner verify
echo "MARKETPLACE_HARDHAT_RPC=http://host.docker.internal:8545" >> infra/.env

docker compose -f infra/docker-compose.yml --profile ssi-wallet --profile mv-bridge up --build -d
sleep 5

# Is the fifth VC type exposed?
curl -s http://192.168.68.53:8080/.well-known/openid-credential-issuer \
  | python3 -m json.tool | grep -A1 SellerVC
# → "vct": "https://iw3ip.example/credentials/SellerVC/v1"
```

---

## Step S1. Issue a SellerVC

### What to check

- You pass `seller_id` and `licensed_datasets` (a whitelist of dataset_id values) in the query, and
  the wallet receives a SellerVC
- Datasets other than those written in `licensed_datasets` are rejected later

### Steps

In a PC browser:

```
http://192.168.68.53:8080/issuer/offer?type=SellerVC&seller_id=ertl-seller-001&licensed_datasets=home/env/temperature,home/env/humidity
```

Scan the QR code with the iPhone wallet → approve **"IW3IP Seller Credential"** → receive.

### Expected result

- "IW3IP Seller Credential" is added to the wallet's VC list
- claim:
    - `seller_id`: `ertl-seller-001`
    - `licensed_datasets`: `["home/env/temperature", "home/env/humidity"]`
    - `subject_id`: `did:jwk:...`

---

## Step S2. Present the SellerVC and obtain a SellerToken

### What to check

- The presentation yields a **SellerToken** (a token separate from
  PolicyToken / ViewerToken / ServiceToken)
- The TTL is **24 hours (86400 seconds)**
- The response contains `licensed_datasets` as is

### Steps

In a PC browser:

```
http://192.168.68.53:8080/verifier/request?vc_kind=SellerVC
```

The `dataset_id` parameter is not needed (a SellerVC does not depend on a specific dataset).
QR code → select the **SellerVC** in the wallet and present it.

### Expected output

```bash
PUB=$(docker ps -qf name=publisher)
docker logs $PUB 2>&1 | grep "seller_token_issued" | tail -1
```

→

```
seller_token_issued jti=... token=... seller_id=ertl-seller-001 licensed=['home/env/temperature', 'home/env/humidity'] ttl=86400s
```

Put the token in a variable.

```bash
SELLER=$(docker logs $PUB 2>&1 | grep "seller_token_issued" | tail -1 | sed -E 's/.*token=([^ ]+).*/\1/')
echo "SELLER=$SELLER"
```

---

## Step S3. Deploy a Merchandise and register it in IoTMarket

### What to check

- Deploy a Merchandise in the Hardhat console and call `IoTMarket.registerMerchandise()`
- At deploy time, put `dataset_id=home/env/temperature` in `additionalInfo`
  (the same format as Stage 6)

### Steps

Terminal B (Hardhat console):

```bash
cd ~/program/Blockchain_IoT_Marketplace/iot-market
npx hardhat console --network localhost
```

```javascript
// use seller = Account #1 (iotOwner)
const [_, iotOwner] = await ethers.getSigners();
console.log("seller:", await iotOwner.getAddress());

// Register a PubKey (only on first use of PubKey)
const pubKey = await ethers.getContractAt("PubKey", "0x5FbDB2315678afecb367f032d93F642f64180aa3", iotOwner);
try { await (await pubKey.registerKey("[seller-key]")).wait(); } catch (e) { /* ignore if already registered */ }

// Deploy a Merchandise (include dataset_id in additionalInfo)
const dataHash = "0x" + "5".repeat(64);
const merch = await (await ethers.getContractFactory("Merchandise", iotOwner)).deploy(
  ethers.parseEther("0.02"),
  dataHash,
  pubKey,
  [],   // deniedBuyers
  ["fileType", "dataSize", "dataset_id"],
  ["json", "1MB", "home/env/temperature"]
);
const deployReceipt = await merch.deploymentTransaction().wait();
const merchandiseAddress = await merch.getAddress();
console.log("merchandise:", merchandiseAddress);
console.log("deploy tx:", deployReceipt.hash);

// Register in IoTMarket
const market = await ethers.getContractAt("IoTMarket", "0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512", iotOwner);
const regTx = await market.registerMerchandise(merch);
const regReceipt = await regTx.wait();
console.log("register tx:", regReceipt.hash);
```

### Note down

Note down the four values shown in the console: `seller`, `merchandise`, `deploy tx`, and `register tx`.

---

## Step S4. Bind the seller on the publisher with `/marketplace/register`

### What to check

- `/marketplace/register` succeeds with the SellerToken as Bearer
- Only a dataset_id contained in `licensed_datasets` passes
- (Optional) When MARKETPLACE_HARDHAT_RPC is enabled, Merchandise.getOwner() is verified
- A `marketplace/seller_registered` row appears in the audit log

### Steps

```bash
SELLER_ETH=<seller address from Step S3>
MERCH=<merchandise address from Step S3>
TX_HASH=<register tx hash from Step S3>

curl -s -X POST http://192.168.68.53:8080/marketplace/register \
  -H "Authorization: Bearer $SELLER" \
  -H "Content-Type: application/json" \
  -d "{
    \"merchandise_address\": \"$MERCH\",
    \"seller_eth_addr\": \"$SELLER_ETH\",
    \"tx_hash\": \"$TX_HASH\",
    \"dataset_id\": \"home/env/temperature\"
  }" | python3 -m json.tool
```

### Expected output

```json
{
  "registered": true,
  "merchandise_address": "0x...",
  "seller_did": "did:jwk:...",
  "dataset_id": "home/env/temperature",
  "tx_hash": "0x...",
  "register_count": 1,
  "owner_verify": "skipped"
}
```

Values of `owner_verify`:

- `skipped`: `MARKETPLACE_HARDHAT_RPC` is not set (for testing / development)
- `verified`: getOwner() matches seller_eth_addr on-chain
- `rpc_failed`: the RPC cannot be reached, and the call returns 502 (`fail-closed`)

### audit log

```bash
curl -s 'http://192.168.68.53:8080/audit/logs?limit=3' | python3 -m json.tool
```

→ Among the most recent rows:

```json
{
  "raw_topic": "marketplace/seller_registered",
  "subject_did": "did:jwk:...",
  "purpose": "register",
  "reason": "seller_register:jti=...:merchandise=0x...:eth=0x...:tx=0x...:owner_verify=skipped",
  "holder_did": "did:jwk:..."
}
```

---

## Step S5. The `seller_did` seen by the Buyer

### What to check

- When the Buyer calls `/platform/data?merchandise=<addr>` via a PurchaseViewerVC,
  the response contains **`seller_did`**

### Steps

Follow the procedure of the [Stage 5 hands-on](marketplace-vc-bridge.md): purchase with MetaMask → receive in the wallet
→ obtain a ViewerToken → fetch.

```bash
TOKEN=<the buyer's ViewerToken>
curl -s -H "Authorization: Bearer $TOKEN" \
  "http://192.168.68.53:8080/platform/data?merchandise=$MERCH" | python3 -m json.tool
```

### Expected output

```json
{
  "dataset_id": "home/env/temperature",
  "count": N,
  "read_count": 1,
  "seller_did": "did:jwk:...(the seller's DID)",
  "rows": [...]
}
```

`seller_did` is the value bound in Step S4. For a Merchandise that has only gone through Stage 5/6
(`/marketplace/register` has not been called), `"unknown"` is returned.

---

## Step S6. Register from the `/seller` page of iot-market-ui

### What to check

- `/marketplace/register` can also be called from the Web UI
- The three-step structure of the form (issue → present → register) is self-contained from the seller's point of view

### Steps

With iot-market-ui already running:

```
http://192.168.68.53:5173/seller
```

Procedure in the UI:

1. **Step 1**: enter `seller_id` and `licensed_datasets`, then
   "Open /issuer/offer" → receive in the wallet
2. **Step 2**: "Open /verifier/request" → present the SellerVC in the wallet →
   copy the `SellerToken` from the publisher log
3. **Step 3**: paste the token obtained above and the merchandise/eth/tx from Step S3,
   select the `dataset_id`, and press "Register"

On success, the same JSON response as in Step S4 is shown at the bottom of the screen.

---

## Completion matrix

| Step | Check item | Example output (values obtained in verification on real devices) |
| --- | --- | --- |
| S0 | The fifth type, SellerVC, appears in the metadata | `"vct": "https://iw3ip.example/credentials/SellerVC/v1"` |
| S1 | SellerVC in the wallet | `seller_id: "ertl-seller-001"` and `licensed_datasets: ["home/env/temperature", "home/env/humidity"]` in the claims |
| S2 | SellerToken issued | `seller_token_issued jti=... seller_id=ertl-seller-001 licensed=['home/env/temperature', 'home/env/humidity'] ttl=86400s` |
| S3 | Merchandise + IoTMarket registration | two transactions: deploy tx + register tx |
| S4 | `/marketplace/register` succeeds | `{ "registered": true, "seller_did": "did:jwk:...", "register_count": <cumulative count>, "owner_verify": "verified" / "skipped" }` |
| S4 | audit log | `raw_topic=marketplace/seller_registered`, `reason=seller_register:jti=...:merchandise=0x...:eth=0x...:tx=0x...:owner_verify=...` |
| S5 | seller_did shown on the buyer side | `{ "dataset_id": "home/env/temperature", "seller_did": "did:jwk:...", ... }` |
| S6 | Registration also works through the UI | the response JSON is shown at the bottom of the screen after the `/seller` form is submitted |

**Completion condition for Stage 7**: the response JSON of `/platform/data?merchandise=<addr>`
**contains a `seller_did` field with a did:jwk: value**. This lets you check a record,
backed by a VC, of which seller sold the data.

---

## Troubleshooting

### A. 403 with `seller_token_dataset_not_licensed`

**Cause**: You passed to `/marketplace/register` a dataset that is not contained in the
`licensed_datasets` of the SellerVC.

**Fix**: Add more entries to `licensed_datasets` in Step S1, or issue a separate
SellerVC for the target dataset.

### B. `chain_rpc_failed` (502) when `MARKETPLACE_HARDHAT_RPC` is enabled

**Cause**: The publisher container cannot reach host.docker.internal:8545.

**Fix**: Check that the Hardhat node is running on the host with `--hostname 0.0.0.0`.
Add the `extra_hosts` setting to the publisher service in `infra/docker-compose.yml`,
and then run `docker compose up -d --build publisher` again.

### C. `owner_mismatch` (403)

**Cause**: The eth address of the SellerVC holder differs from that of Merchandise.getOwner().
This happens when the Hardhat account used for the deploy differs from the eth account that obtained the SellerVC.

**Fix**: Redo receiving the SellerVC and deploying the Merchandise with the same Hardhat signer
(for example, Account #1 = `iotOwner`).

### D. `try { ... } catch (e) { ... }` causes a SyntaxError in the Hardhat console

```
Uncaught SyntaxError: missing ) after argument list
```

**Cause**: The Hardhat console (Node REPL) evaluates statements one line at a time, so pasting a
`try`/`catch` that spans multiple lines causes a syntax error on the first line.

**Fix**: Write it on one line. Example:

```javascript
let r; try { r = await (await pubKey.registerKey("[buyer-key]")).wait(); console.log("ok:", r.status); } catch (e) { console.log("err:", e.message); }
```

### E. After restarting the Hardhat node, `getState()` and similar calls return 0x and fail with `BAD_DATA`

```
ERROR: could not decode result data (value="0x", info={ "method": "getState", ...
```

**Cause**: Stopping the Hardhat node with `Ctrl+C` or similar **erases all
on-chain state**. The already deployed PubKey / IoTMarket / Merchandise also
lose their code. An address for which `provider.getCode(addr)` returns `"0x"` (= length 2)
has no contract.

**Fix**:

```javascript
// Check whether the main addresses have code
const c1 = await ethers.provider.getCode("0x5FbDB2315678afecb367f032d93F642f64180aa3");  // PubKey
const c2 = await ethers.provider.getCode("0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512");  // IoTMarket
console.log({ pubKey: c1.length, ioTMarket: c2.length });
// If 2 is printed, the code is gone
```

If it is gone, redeploy with `npx hardhat run scripts/deployMerchandiseWithIoTMarket.ts --network localhost`.

Note: on the chain after a node restart, a Merchandise that you deployed manually in the previous session
(other than the deterministic addresses of the script) is not restored. **After a chain reset,
it is safe to use only the five contracts deployed by the script.**

### F. After redeploying the chain, a purchase by Account #2 fails with `function returned an unexpected amount of data`

**Cause**: The PubKey contract was redeployed and its state was reset, so
the public key registration of the buyer (Account #2) is gone. The
`i_pubKey.getPubKey(tx.origin)` called inside Merchandise.purchase() returns an empty response, and Solidity
fails to decode the `string`.

**Fix**: Register the PubKey again.

```javascript
const [_, __, buyer] = await ethers.getSigners();
const pk = await ethers.getContractAt("PubKey", "0x5FbDB2315678afecb367f032d93F642f64180aa3", buyer);
let r; try { r = await (await pk.registerKey("[buyer-key]")).wait(); console.log("ok:", r.status); } catch (e) { console.log("err:", e.message); }
```

Register in the same way when you use the seller (Account #1).

### G. No new Purchase event appears in the bridge log / the wallet gets 400 at `/issuer/token`

**Cause**: The bridge listener polls only blocks after the block cursor obtained at
startup. Restarting Hardhat resets the block number to 0,
so the bridge can see only "past" blocks. Alternatively, the wallet tries to
reuse a pre_authorized_code issued on the old chain, and
the publisher treats it as consumed and returns 400.

**Fix**: Force-recreate the bridge container to reset the cursor:

```bash
docker compose -f infra/docker-compose.yml --profile mv-bridge up -d --force-recreate bridge
sleep 5
docker logs iw3ip-mv-bridge 2>&1 | tail -5
# → bridge: starting poll from block <current number>
```

After that, running `Merchandise.purchase()` again lets the bridge capture the new Purchase event.
Discard the old deeplink and take a new one from the new `bridge: claim ok` line.

### H. Other issues

See also [Troubleshooting in Stage 5](marketplace-vc-bridge.md#troubleshooting)
(MetaMask chainId cache, `BRIDGE_PUBLIC_PUBLISHER_URL` not set, etc.).

---

## Limits / future work

- **On-chain guard**: currently `IoTMarket.registerMerchandise()` itself
  passes even without publisher verification. An on-chain guard is future work (Stage 8 or later)
- **EIP-712 signatures**: the binding between the seller_did obtained via the SellerVC and the eth address
  relies on trust by the publisher (the same MVP level as eth_did_bound in Stage 5/6)
- **Where to declare the dataset_id**: Stage 6 case B made it possible to read the
  `dataset_id` value from `additionalInfo`, but `/marketplace/register` depends on the client
  POST. One future option is for the publisher itself to call `getAllAdditionalInfo()` and
  cross-check the value

## Related

- [SellerVC design spec](../design/seller-vc-spec.md)
- [Marketplace × Wallet bridge (Stage 5)](marketplace-vc-bridge.md)
- [Marketplace VC end-to-end (Stage 6)](marketplace-vc-end-to-end.md)
- [SSI Service (Stage 4 prep)](ha-ssi-service.md)
- [SSI Wallet (Stage 1)](ha-ssi-wallet.md)
