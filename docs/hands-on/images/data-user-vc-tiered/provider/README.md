# Stage T `/provider` page screenshots

Holds the §11.8 real-device validation screenshots referenced from
`docs/hands-on/data-user-vc-tiered.md` (and `.en.md`).

## Naming convention

`<scenario>-<browser>-<step>.png` where:

- `<scenario>` ∈ `A` (iPhone capture) / `B` (Chrome record) / `C` (Firefox record) / `D` (Safari record)
- `<browser>` ∈ `iphone` / `chrome` / `firefox` / `safari`
- `<step>` ∈ `permission`, `recording`, `uploaded`, `publish-ok`, `devtools-codec`, `capture-sheet`, `after-record`, `after-publish`

Examples:

- `A-iphone-capture-sheet.png`
- `B-chrome-recording.png`
- `C-firefox-devtools-codec.png`
- `D-safari-publish-ok.png`

## Resolution / format guidance

- PNG, max ~1600 px wide (mkdocs renders fine on retina + saves bandwidth)
- For DevTools-codec screenshots, crop to the relevant pane only — no full-screen
- Strip any wallet seed phrases / private DIDs / personal LAN hostnames before committing

## Linking

Reference from the markdown like:

```markdown
![B Chrome recording](images/data-user-vc-tiered/provider/B-chrome-recording.png)
```

Path is relative to `docs/hands-on/data-user-vc-tiered.md`.
