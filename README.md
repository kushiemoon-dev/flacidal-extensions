# FLACidal Extensions

Default extension registry for [FLACidal](https://github.com/kushiemoon-dev/FLACidal) and [FLACidal-Mobile](https://github.com/kushiemoon-dev/FLACidal-Mobile).

`index.json` is fetched by the app's "Browse" tab. No extensions published yet — coming soon.

## Manifest format

Two related-but-different JSON shapes are involved:

1. **`extension.json`** — the full manifest, bundled inside an extension's downloadable zip.
   This is what FLACidal-Core's `ExtensionManifest` struct (plus the `SourceExtCfg`,
   `MetadataExtCfg`, and `AuthField` types it embeds) unmarshals on install.
2. **An `index.json` entry** — a deliberately minimal listing shape used by this registry,
   so the app's Browse tab can show a catalog without downloading every extension's zip.

Both are formalized as JSON Schema (draft 2020-12) in [`schema/`](./schema):
[`extension-manifest.schema.json`](./schema/extension-manifest.schema.json) validates a single
`extension.json`, and [`registry-entry.schema.json`](./schema/registry-entry.schema.json)
validates `index.json` as a whole (one document = one array of entries).

### `extension.json` (full manifest)

Field names below are grouped by the Go struct that carries them. Some names repeat across
groups on purpose (e.g. `name` on both the manifest and `sourceConfig`, `baseUrl` on both
`sourceConfig` and `metadataConfig`) — each occurrence is independent, not a shared value.

**`ExtensionManifest`** (top level)

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | string | yes | Unique, stable identifier for the extension. Used as its install directory name. | `"acme-hires"` |
| `name` | string | yes | Human-readable display name shown in the app. | `"Acme Hi-Res"` |
| `version` | string | yes | The extension's own semver, for this zip's contents. Distinct from `index.json`'s `latestVersion` — see below. | `"1.3.0"` |
| `minAppVersion` | string | no | Minimum FLACidal app version required. Treated as "no minimum" when empty/omitted — not required by this schema, even though the Go struct has no `omitempty` on it (that only affects marshal-out behavior, not what Core accepts on install). | `"2.5.0"` |
| `author` | string | yes | Extension author's name or handle. | `"acme-labs"` |
| `description` | string | no | Short summary of what the extension does. | `"Streams and downloads Acme's hi-res FLAC catalog."` |
| `category` | string | no | Free-form grouping for the Browse tab. Suggested values (not enforced by Core): `download`, `metadata`, `lyrics`, `utility`. | `"download"` |
| `capabilities` | string[] | yes | What the extension can do. Suggested values (not enforced by Core): `source`, `metadata`, `resolver`. Determines whether `sourceConfig`/`metadataConfig` are expected. | `["source", "metadata"]` |
| `permissions` | string[] | yes | Permissions requested, shown to the user before install. Suggested value (not enforced by Core): `network`. | `["network"]` |
| `canDownload` | bool | no | Whether this extension can act as a download source. Pairs with `sourceConfig` and `downloadPriority`. | `true` |
| `downloadPriority` | int | no | Order download-capable extensions are tried in. Lower = tried first, `0` = disabled. | `10` |
| `sourceConfig` | object | no | Source (download) configuration — see `SourceExtCfg` below. Present when `capabilities` includes `source`. | see below |
| `canEnrichMetadata` | bool | no | Whether this extension can act as a metadata enrichment source. Pairs with `metadataConfig` and `metadataPriority`. | `true` |
| `metadataPriority` | int | no | Order metadata-capable extensions are tried in. Lower = tried first, `0` = disabled. | `5` |
| `metadataConfig` | object | no | Metadata enrichment configuration — see `MetadataExtCfg` below. Present when `capabilities` includes `metadata`. | see below |
| `authFields` | array | no | Credential fields the app should prompt the user for — see `AuthField` below. | see below |

**`SourceExtCfg`** (`sourceConfig`)

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `name` | string | yes | Internal identifier for this source (distinct from the manifest's top-level `name`). | `"acme"` |
| `displayName` | string | yes | Human-readable source name shown in the app. | `"Acme Music"` |
| `urlPattern` | string | yes | Regex matched against a shared/pasted URL's host, to route it to this extension. | `"acme-music\\.com"` |
| `baseUrl` | string | yes | Base URL the endpoint fields below are relative to. | `"https://api.acme-music.com/v1"` |
| `trackEndpoint` | string | no | Endpoint for fetching a single track. | `"/track/{id}"` |
| `albumEndpoint` | string | no | Endpoint for fetching an album. | `"/album/{id}"` |
| `playlistEndpoint` | string | no | Endpoint for fetching a playlist. | `"/playlist/{id}"` |
| `searchEndpoint` | string | no | Endpoint for search queries. | `"/search"` |
| `streamEndpoint` | string | no | Endpoint for resolving a track's streamable/downloadable URL. | `"/track/{id}/stream"` |
| `trackMapping` | map[string]string | no | Maps FLACidal track fields to dot-separated JSON paths in this source's API response. | `{"title": "track.title"}` |
| `albumMapping` | map[string]string | no | Maps FLACidal album fields to dot-separated JSON paths in this source's API response. | `{"title": "album.title"}` |
| `streamUrlField` | string | no | Dot-separated JSON path to the streamable URL within the `streamEndpoint` response. | `"data.url"` |

**`MetadataExtCfg`** (`metadataConfig`)

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `baseUrl` | string | yes | Base URL the `lookupEndpoint` field is relative to. | `"https://api.acme-music.com/v1"` |
| `lookupEndpoint` | string | yes | Endpoint for looking up metadata by ISRC, templated with `{isrc}`. | `"/metadata/lookup?isrc={isrc}"` |
| `fieldMapping` | map[string]string | no | Maps a track-metadata field name to a dot-separated JSON path in the lookup response. | `{"genre": "result.genres.0.name"}` |

**`AuthField`** (entries of `authFields`)

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `key` | string | yes | Key this credential is stored/passed under (e.g. as a request header name). | `"apiKey"` |
| `label` | string | yes | Label shown to the user in the credential entry form. | `"API Key"` |
| `type` | string | yes | Input type hint for the credential form. Suggested values (not enforced by Core): `text`, `password`. | `"password"` |
| `required` | bool | yes | Whether the user must fill in this field before the extension can be used. | `true` |

Full example (`extension.json` inside the zip):

```json
{
  "id": "acme-hires",
  "name": "Acme Hi-Res",
  "version": "1.3.0",
  "minAppVersion": "2.5.0",
  "author": "acme-labs",
  "description": "Streams and downloads Acme's hi-res FLAC catalog, and enriches tracks with Acme's editorial metadata.",
  "category": "download",
  "capabilities": ["source", "metadata"],
  "permissions": ["network"],
  "canDownload": true,
  "downloadPriority": 10,
  "sourceConfig": {
    "name": "acme",
    "displayName": "Acme Music",
    "urlPattern": "acme-music\\.com",
    "baseUrl": "https://api.acme-music.com/v1",
    "trackEndpoint": "/track/{id}",
    "albumEndpoint": "/album/{id}",
    "playlistEndpoint": "/playlist/{id}",
    "searchEndpoint": "/search",
    "streamEndpoint": "/track/{id}/stream",
    "trackMapping": { "title": "track.title", "artist": "track.artist.name" },
    "albumMapping": { "title": "album.title", "artist": "album.artist.name" },
    "streamUrlField": "data.url"
  },
  "canEnrichMetadata": true,
  "metadataPriority": 5,
  "metadataConfig": {
    "baseUrl": "https://api.acme-music.com/v1",
    "lookupEndpoint": "/metadata/lookup?isrc={isrc}",
    "fieldMapping": { "genre": "result.genres.0.name", "label": "result.label.name" }
  },
  "authFields": [
    { "key": "apiKey", "label": "API Key", "type": "password", "required": true }
  ]
}
```

### `index.json` entry (registry listing)

Each entry in `index.json` is intentionally minimal — it is **not** the full manifest above.
It exists so the Browse tab can list and preview extensions without fetching every zip.

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | string | yes | Extension identifier, matching `id` inside the extension's own `extension.json`. | `"acme-hires"` |
| `name` | string | yes | Display name shown in the Browse tab. | `"Acme Hi-Res"` |
| `description` | string | no | Short summary shown in the Browse tab. | `"Streams and downloads Acme's hi-res FLAC catalog."` |
| `category` | string | no | Free-form grouping used in the Browse tab. | `"download"` |
| `latestVersion` | string | yes | Listing-only: the version currently published at `downloadURL`. | `"1.3.0"` |
| `downloadURL` | string | yes | Listing-only: direct download link for the extension's zip. | `"https://github.com/acme-labs/acme-hires-extension/releases/download/v1.3.0/acme-hires.zip"` |
| `permissions` | string[] | no | Preview of the permissions requested, shown before install (a copy of the manifest's `permissions`). | `["network"]` |

`latestVersion` and `downloadURL` are **listing-only** fields, distinct from `version`, which
lives inside the manifest (`extension.json`) itself and is the extension author's own semver for
the zip's contents. `latestVersion` is maintained by whoever publishes/updates this registry
entry (e.g. when bumping to a new release) — it is not read automatically from the zip. The name
divergence (`latestVersion` here vs. `version` in the manifest) is an existing fact of the app
today, not an inconsistency to fix in this repo.

Example entry:

```json
{
  "id": "acme-hires",
  "name": "Acme Hi-Res",
  "description": "Streams and downloads Acme's hi-res FLAC catalog, and enriches tracks with Acme's editorial metadata.",
  "category": "download",
  "latestVersion": "1.3.0",
  "downloadURL": "https://github.com/acme-labs/acme-hires-extension/releases/download/v1.3.0/acme-hires.zip",
  "permissions": ["network"]
}
```

`index.json` itself is a JSON array of these entries (currently `[]` — no extensions published
yet).

## Contributing

Before opening a pull request, validate your changes locally:

```bash
# Validate index.json against the registry schema
npx ajv-cli validate --spec=draft2020 --strict=false -s schema/registry-entry.schema.json -d index.json

# Validate all extension manifests
npx ajv-cli validate --spec=draft2020 --strict=false -s schema/extension-manifest.schema.json -d "extensions/*/extension.json"
```

These commands will run automatically on every pull request targeting `main` via GitHub Actions.
