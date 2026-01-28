# STAC API - Merkle Tree Verification Extension

- **Title:** Merkle Tree Verification
- **Conformance Classes:**
  - `https://api.stacspec.org/v1.0.0/core` (required)
  - `https://api.stacspec.org/v1.0.0-beta.1/merkle-verification` (required)
- **Scope:** STAC API - Item Search & Collections
- **Extension Maturity Classification:** Proposal
- **Dependencies:**
  - [STAC API - Core](https://github.com/radiantearth/stac-api-spec/blob/main/core)
  - [STAC Merkle Tree Specification](https://github.com/stacchain/merkle-tree)
- **Owner**: @jonhealy1

## Introduction

This extension adds a **Data Integrity Layer** to the STAC API. It enables users to cryptographically verify that the geospatial assets they retrieve have not been tampered with, corrupted (bit rot), or swapped since they were originally published.

It introduces standard locations for **Merkle Roots** (at the Collection level) and **Inclusion Proofs** (at the Item level), along with a convenience endpoint for server-side verification.

### The Trust Model
In a standard STAC API, trust is **implied** ("I trust the URL I am visiting"). In a Merkle-enabled API, trust is **derived**:
1.  **The Root:** A cryptographic commitment to the entire dataset state.
2.  **The Proof:** A mathematical path linking a specific Item to that Root.
3.  **The Asset:** The file itself, hashed and checked against the Item metadata.

This architecture allows for **"Verify-then-Trust"** workflows, critical for high-assurance domains like Carbon Markets, Disaster Response, and Parametric Insurance.

## Endpoints

This extension introduces a new utility endpoint to simplify verification for lightweight clients.

| Method | URI | Description |
| :--- | :--- | :--- |
| `GET` | `/collections/{collectionId}/items/{itemId}/verify` | **Server-Side Verification.** Checks the integrity of a specific item against its collection's Merkle Root. |

## Data Model Extensions

To support verification, this extension mandates specific fields in the standard STAC API responses.

### 1. Collection Object Extension
The Collection metadata acts as the "Anchor of Truth."

| Field | Type | Description |
| :--- | :--- | :--- |
| `merkle:root` | `string` | **Required.** The Hex-encoded Merkle Root Hash of the entire collection. This is the value that should be anchored on a blockchain or public bulletin board. |
| `merkle:root_method` | `string` | **Recommended.** The hashing algorithm used (e.g., `sha256`). Defaults to `sha256` if omitted. |

### 2. Item Object Extension
The Item metadata acts as the "Leaf" of the tree.

| Field | Type | Description |
| :--- | :--- | :--- |
| `merkle:object_hash` | `string` | **Required.** The hash of the canonical JSON representation of this Item (as calculated by the STAC Merkle CLI). |

### 3. Link Relations
The API must provide a way to retrieve the Proof file needed to connect the Item to the Root.

- `rel="merkle-proof"`: **Required.** Points to a JSON file containing the Merkle Inclusion Proof (siblings path) for this specific item.
- `rel="merkle-source"`: **Optional.** Points to the raw source data used to generate the hash, if different from the Item itself.

## Verification Behavior (`GET .../verify`)

While clients *can* (and should) verify proofs locally using the fields above, the `/verify` endpoint provides a standardized way for the API to assert its own integrity.

**Request:**
`GET /collections/sentinel-2-l2a/items/S2A_tile_20240101/verify`

**Logic:**
1.  **Fetch Item:** Retrieves the current item metadata from the database.
2.  **Fetch Proof:** Retrieves the `merkle-proof` file (from S3/Storage).
3.  **Calculate:**
    * Computes `Hash(Item)`.
    * Traverses the path in the Proof using the computed hash.
    * Derives a candidate Root.
4.  **Compare:** Checks if `Candidate Root == Collection.merkle:root`.

**Response (200 OK):**
Returns a status object indicating the mathematical validity of the item.

```json
{
  "verified": true,
  "item_id": "S2A_tile_20240101",
  "merkle_root": "a1b2c3d4e5f6...",
  "anchored_status": {
    "on_chain": false,
    "timestamp": "2024-01-28T12:00:00Z"
  }
}