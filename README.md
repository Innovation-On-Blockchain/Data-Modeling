# Ethereum AML Data Modeling

Data formatting pipeline that transforms raw Ethereum transaction data into GNN-ready input files for node-level sanctions classification.

This repository sits between the [Data-Preprocessing-For-AML](https://github.com/Innovation-On-Blockchain/Data-Preprocessing-For-AML) Rust pipeline (which collects and aggregates on-chain data) and the GNN training stage (which adapts IBM's [Multi-GNN](https://github.com/IBM/Multi-GNN) for node-level classification).

---

## What This Repo Does

The Rust preprocessing pipeline outputs raw Parquet files with string addresses, ISO timestamps, and wei-denominated values. The GNN training code expects integer-indexed node IDs, relative timestamps, ETH-denominated amounts, and explicit train/val splits.

This notebook bridges that gap:

1. Loads raw transactions and OFAC sanction labels from the Rust pipeline
2. Filters out smart contract addresses using the pipeline's `nodes.parquet` metadata
3. Maps Ethereum addresses to contiguous integer IDs (0-indexed)
4. Converts timestamps to relative seconds and values from wei to ETH
5. Labels edges as illicit if either endpoint is OFAC-sanctioned
6. Produces three output files in the exact format IBM Multi-GNN expects
7. Creates an 80/20 temporal train/validation split

---

## Pipeline Overview

```
Rust Pipeline Outputs                   This Notebook                       GNN Training Inputs
(string addresses, ISO timestamps)      (Modeling_and_preprocessing.ipynb)   (integer IDs, relative timestamps)

  transactions_raw.parquet ──┐
        (90.7M rows)         │
                             ├──► Load ──► Filter Contracts ──► Map IDs ──► Convert ──► Label ──► Split
  labels.parquet ────────────┤
        (82 addresses)       │                                        │
                             │                                        ├──► formatted_transactions.parquet
  nodes.parquet ─────────────┘                                        ├──► node_labels.parquet
     (33.9M nodes,                                                    └──► data_splits.json
      is_contract flag)
```

---

## Input Files

These are produced by the [Data-Preprocessing-For-AML](https://github.com/Innovation-On-Blockchain/Data-Preprocessing-For-AML) Rust pipeline and are **not** included in this repository.

| File | Pipeline Stage | Location | Description |
|------|---------------|----------|-------------|
| `transactions_raw.parquet` | Stage 2 (fetch-transactions) | `data/raw/` | 90,728,163 raw transaction records |
| `labels.parquet` | Stage 1 (fetch-labels) | `data/labels/` | 82 OFAC-sanctioned Ethereum addresses |
| `nodes.parquet` | Stage 4 (build-nodes) | `data/processed/` | 33,895,542 nodes with `is_contract`, `degree_in`, `degree_out`, `node_type` |

### Input Schemas

**`transactions_raw.parquet`**

| Column | Type | Example |
|--------|------|---------|
| `tx_hash` | string | `0xabc123...` |
| `block_timestamp` | string (ISO-8601) | `2023-01-15T12:30:45+00:00` |
| `from_address` | string | `0x1234...abcd` |
| `to_address` | string | `0x5678...efgh` |
| `value_wei` | string | `1000000000000000000` |

**`labels.parquet`**

| Column | Type | Description |
|--------|------|-------------|
| `address` | string | EIP-55 checksummed Ethereum address |
| `label` | int | 1 = sanctioned |
| `label_source` | string | `OFAC_SDN` |
| `source_url` | string | Treasury URL |
| `retrieved_at` | string | UTC timestamp |

**`nodes.parquet`**

| Column | Type | Description |
|--------|------|-------------|
| `address` | string | Ethereum address |
| `is_contract` | bool | `True` if smart contract, `False` if EOA |
| `degree_in` | int64 | Unique incoming counterparties |
| `degree_out` | int64 | Unique outgoing counterparties |
| `node_type` | string | `eoa`, `contract`, or `hub` (top 0.1% by degree) |

---

## Processing Steps

The notebook (`Modeling_and_preprocessing.ipynb`) performs 8 sequential steps:

### Step 1: Filter Contract Addresses

Loads `nodes.parquet` and extracts all addresses where `is_contract == True` (721,891 contracts identified by the Rust pipeline via `eth_getCode` RPC calls and BigQuery lookup). Removes every transaction where **either** `from_address` or `to_address` is a contract.

This ensures the output graph contains only EOA-to-EOA transactions. Contract addresses (DEX routers, bridges, multisig wallets) are structural intermediaries that would dominate the graph topology and obscure the signal from sanctioned EOA addresses.

### Step 2: Create Address-to-ID Mapping

Collects all unique addresses from the remaining (EOA-only) transactions, sorts them deterministically, and assigns each a numeric ID starting from 0. This produces contiguous, 0-indexed integer IDs compatible with PyTorch Geometric tensor indexing.

### Step 3: Convert Timestamps

Parses ISO-8601 `block_timestamp` strings to datetime objects, sorts the dataframe chronologically, then converts to relative integer seconds (earliest transaction = 0). The temporal ordering is critical — the train/val split (Step 8) depends on it.

### Step 4: Convert Values (Wei to ETH)

Converts `value_wei` strings (1 ETH = 10^18 wei) to float ETH values. Stores as both `Amount Sent` and `Amount Received` (they are equal for direct ETH transfers — this dual-column format matches the IBM Multi-GNN schema which was designed for multi-currency synthetic data).

### Step 5: Label Edges

For each transaction: `Is Laundering = 1` if **either** the sender or receiver is in the OFAC sanctioned set, else `0`. Edges between two clean addresses are kept as negative examples — they provide the GNN with examples of normal graph structure.

### Step 6: Build Formatted Transactions File

Assembles the final edge list with columns in IBM Multi-GNN order. Three placeholder columns (`Sent Currency`, `Received Currency`, `Payment Format`) are set to constant `1` for format compatibility — they carry no information on single-chain ETH-only data.

### Step 7: Build Node Labels File

For each unique address in the address-to-ID mapping: `is_sanctioned = 1` if the address is in the OFAC sanctioned set, else `0`. First column is the node ID, last column is the binary label — matching the spec expected by the GNN training code.

### Step 8: Create Train/Validation Split

Temporal 80/20 split on the chronologically sorted edge list. Train = first 80% of edges (earliest transactions), Val = last 20% (latest transactions). This prevents temporal data leakage. No test split is included — in the transfer learning setup, the test domain is a different blockchain.

---

## Output Files

Three files are written to the output directory:

### `formatted_transactions.parquet`

Edge list with features. Each row is one transaction.

| Column | Type | Description |
|--------|------|-------------|
| `EdgeID` | int64 | Sequential 0-indexed edge identifier |
| `from_id` | int64 | Sender node ID (integer-mapped) |
| `to_id` | int64 | Receiver node ID (integer-mapped) |
| `Timestamp` | int64 | Relative seconds from earliest transaction |
| `Amount Sent` | float64 | ETH value transferred |
| `Sent Currency` | int64 | Always 1 (format placeholder) |
| `Amount Received` | float64 | ETH value transferred (= Amount Sent) |
| `Received Currency` | int64 | Always 1 (format placeholder) |
| `Payment Format` | int64 | Always 1 (format placeholder) |
| `Is Laundering` | int64 | 1 if either endpoint is sanctioned, else 0 |

### `node_labels.parquet`

One row per unique EOA address in the graph.

| Column | Type | Description |
|--------|------|-------------|
| `node_id` | int64 | Contiguous 0-indexed node identifier |
| `is_sanctioned` | int64 | 1 = OFAC-sanctioned, 0 = not sanctioned |

### `data_splits.json`

```json
{
  "train_edge_ids": [0, 1, 2, ...],
  "val_edge_ids":   [N, N+1, ...]
}
```

Edge IDs reference the `EdgeID` column in `formatted_transactions.parquet`. The split is temporal — train edges come chronologically before validation edges.

---

## Usage

### Google Colab (recommended for large datasets)

1. Open `Modeling_and_preprocessing.ipynb` in Google Colab
2. Upload the three input files to your Google Drive
3. Uncomment the Colab drive mount lines in Cells 1-2, comment out the local path lines
4. Update `DATA_DIR` to point to your Drive folder containing the Rust pipeline's `data/` directory
5. Run all cells sequentially

### Running Locally

Requires ~25 GB RAM for the full 90.7M-row dataset.

```bash
pip install pandas pyarrow
jupyter notebook Modeling_and_preprocessing.ipynb
```

Update the paths in Cell 2:
- `OUTPUT_DIR`: where to write the three output files
- `DATA_DIR`: path to the Rust pipeline's `data/` directory (must contain `raw/transactions_raw.parquet`, `labels/labels.parquet`, and `processed/nodes.parquet`)

---

## Data Characteristics

### Before Contract Filtering (raw input)

| Metric | Value |
|--------|-------|
| Total transactions | 90,728,163 |
| Unique addresses | 33,895,542 |
| Contract addresses | 721,891 |
| EOA addresses | 33,173,651 |
| OFAC-sanctioned addresses | 82 (from labels.parquet) |
| Time span | 2015-08-07 to 2026-02-19 |

### After Contract Filtering (output)

Exact counts depend on the run. Filtering removes all transactions where either endpoint is a contract address. Expected impact:

- Transactions: reduced (edges touching contracts are removed)
- Unique nodes: reduced to EOAs only
- Sanctioned nodes: 75 (7 sanctioned addresses had no EOA-to-EOA transactions)

### Edge Label Distribution

| Label | Description |
|-------|-------------|
| `Is Laundering = 1` | Either sender or receiver is OFAC-sanctioned |
| `Is Laundering = 0` | Neither endpoint is sanctioned |

### Node Label Distribution

| Label | Description |
|-------|-------------|
| `is_sanctioned = 1` | Address is on the OFAC SDN list |
| `is_sanctioned = 0` | Address is not on the OFAC SDN list (unlabeled, not necessarily benign) |

### Notes on Constant Columns

Three edge feature columns (`Sent Currency`, `Received Currency`, `Payment Format`) are always 1. They exist for schema compatibility with the IBM Multi-GNN code, which was designed for a synthetic multi-currency dataset. On real single-chain Ethereum data, these columns carry no information. The GNN engineer may choose to drop them or keep them for format compatibility.

---

## Repository Structure

```
Data-Modeling/
├── README.md                              # This file
├── Modeling_and_preprocessing.ipynb        # Data formatting pipeline
│
├── (user-provided inputs — not in repo)
│   ├── transactions_raw.parquet           # From Rust pipeline Stage 2
│   ├── labels.parquet                     # From Rust pipeline Stage 1
│   └── nodes.parquet                      # From Rust pipeline Stage 4
│
└── (generated outputs — not in repo)
    ├── formatted_transactions.parquet     # GNN-ready edge list
    ├── node_labels.parquet                # GNN-ready node labels
    └── data_splits.json                   # Train/val edge splits
```

---

## Downstream: GNN Training

The output files are consumed by a GNN training pipeline that adapts IBM's [Multi-GNN](https://github.com/IBM/Multi-GNN) from edge-level AML detection to node-level sanctions classification. The adaptation:

- **Keeps** the GINEConv/GATConv message-passing backbone identical
- **Keeps** the edge update MLPs identical
- **Changes** the readout layer from edge-level (concat source + dest + edge features) to node-level (node embedding directly into classification MLP)
- **Changes** the loss from edge-weighted CrossEntropy to node-weighted CrossEntropy

See `nextSteps.md` in the [Data-Preprocessing-For-AML](https://github.com/Innovation-On-Blockchain/Data-Preprocessing-For-AML) repo for the full GNN architecture specification and hyperparameters.

---

## References

- **IBM Multi-GNN:** Cardoso, W., Weber, M., et al. (2022). [Multi-GNN: A Graph Neural Network Framework for Anti-Money Laundering](https://github.com/IBM/Multi-GNN). IBM Research.
- **Data Collection:** [Data-Preprocessing-For-AML](https://github.com/Innovation-On-Blockchain/Data-Preprocessing-For-AML) — Rust pipeline for Ethereum AML data collection and graph construction.
- **OFAC SDN List:** U.S. Department of the Treasury, Office of Foreign Assets Control. [Sanctions List](https://sanctionslist.ofac.treas.gov/Home/SdnList).
