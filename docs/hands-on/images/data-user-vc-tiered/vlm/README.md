# Stage T VLM tier extension screenshots

Holds the §12 hands-on screenshots referenced from
`docs/hands-on/data-user-vc-tiered.md` (and `.en.md`).

## Naming convention

`12-<step>.png` (or `.jpg` for camera output) where `<step>` is one of:

- `pre-blur`     — original image (Tier 3)
- `post-blur`    — Haar-cascade blurred output (Tier 2 image_url_redacted)
- `tier3-keys`   — /platform/data Tier 3 response showing all keys
- `tier2-keys`   — Tier 2 response (no raw image, has redacted + descriptions)
- `tier1-keys`   — Tier 1 (summary) response (description_summary only)
- `desc-diff`    — side-by-side description_full vs description_summary
- `degrade`      — processing_warnings example after `docker stop vlm`

For real-device validation runs, prefix with `V1` … `V7` matching §12.10
scenario IDs:

- `V1-tier3-all-keys.png`
- `V2-tier2-redacted.png`
- `V4-iphone-face-blurred.png`
- `V6-degrade-warnings.png`

## Resolution / format guidance

Same as `../provider/README.md`: PNG, ~1600 px max width, crop to the
relevant pane, scrub any wallet seed phrases / private DIDs / personal
LAN hostnames before committing.
