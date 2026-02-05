# Smart Grid Passive Attack Benchmarking Dataset Generator
**IEEE-aligned HAN / NAN / WAN Communication Layers**

This repository provides a **reproducible dataset generator** for benchmarking  
**passive cyber-physical attacks** in smart grid communication networks.

The generator simulates realistic **time-series communication data** for heterogeneous
grid nodes across **HAN, NAN, and WAN layers**, capturing traffic behavior,
channel dynamics, topology-aware interactions, and **subtle passive attack effects**.

> This repository contains **generator code only**.  
> Generated datasets are written to **local disk** and are intentionally excluded
> from version control.

---

## Key Properties

### Passive attacks only (propagation-level)
- No packet injection, spoofing, replay, or active manipulation.
- Attacks manifest as **channel coherence degradation** and **additional path loss**,
  indirectly affecting **SNR, PER, and latency**.

### Activity-gated labels and perturbations
- Attack labels and perturbations are applied **only when `tx_count > 0`**.
- Prevents unrealistic labeling during silent communication periods.

### Leak-safe dataset splits
- Independent random seeds for **train / validation / test** splits.
- **Burn-in period** for AR / correlated processes (not exported).
- **Z-score normalization fitted on train only**, then applied to val/test.

### Topology-local attacks
- Attacks affect **connected node groups** sampled from an IEEE-inspired adjacency.
- Group size is configurable (default: **3–5 nodes**).

### Shared environment correlation
- Neighboring nodes may share correlated **shadowing and interference trends**
  via global and layer-specific AR(1) processes.

### Tech-specific PER and heavy-tailed latency
- Technology-aware **PER vs. SNR curves**.
- **Heavy-tailed latency spikes** driven by PER and filtered to form burst clusters.

---

## Realism Choice: Option A (Latent Channel Components)

The generator uses internal latent variables:
- `shadow_db` (shadowing)
- `interf_db` (interference)

These are used to compute observable metrics but are **NOT exported as features**.
Only physically plausible measurements (CSI, SNR, PER, latency) are exposed.

---

## Repository Structure

```

.
├── smartgrid_passive_attack_dataset.py
├── README.md
├── requirements.txt
├── .gitignore

```

---

## Generated Outputs

All outputs are written to a local directory (default: `Dataset/`).

### Global files
- `Dataset/metadata.csv`
- `Dataset/metadata_ohe.csv`
- `Dataset/network_adjacency.csv`
- `Dataset/network_weights_W.csv`
- `Dataset/reason_vocab.json`
- `Dataset/attacks_windows_meta.csv`
- `Dataset/config.json`

### Per-client datasets
- `Dataset/clients/client_nodeXX/train.csv`
- `Dataset/clients/client_nodeXX/val.csv`
- `Dataset/clients/client_nodeXX/test.csv`
- `Dataset/clients/client_nodeXX/train_stats.json`

---

## Exported Features

Each CSV contains one row per `(node, t)`.

### Core
- `t`, `node`, `split`
- `attack_label`
- `tx_count`

### Observable channel & traffic
- `csi_amp`
- `snr_db`
- `packet_error`
- `latency`
- `latency_smooth`

### Engineered temporal features
- `csi_entropy`
- `csi_drift`
- `csi_skewness`
- `csi_kurtosis`
- `time_since_last_tx`

### Topology-aware features
- `avg_neighbor_snr_db`
- `avg_neighbor_latency_smooth`
- `avg_neighbor_csi_amp`
- `avg_neighbor_packet_error`
- `rho_like_snr_db`
- `rho_like_latency_smooth`
- `rho_like_csi_amp`
- `rho_like_packet_error`

### Normalized features
- `z_*` versions of all major numeric features  
  (z-score fitted on **train split only**)

---

## Installation

Create a `requirements.txt` file:

```

numpy
pandas

````

Install dependencies:
```bash
pip install -r requirements.txt
````

---

## Usage

Run the dataset generator:

```bash
python smartgrid_passive_attack_dataset.py
```

Upon completion, the script prints attack coverage diagnostics and writes
the dataset to the local output directory.

---

## Configuration

All parameters are defined in the `Config` dataclass, including:

* split lengths (`T_TRAIN`, `T_VAL`, `T_TEST`)
* attack coverage target (`TARGET_ATTACK_FRAC`, enforced on ACTIVE rows)
* attack window sizing and overlap rules
* group size constraints
* technology-specific channel and latency models
* traffic generation logic

The full configuration used for a run is saved to:

```
Dataset/config.json
```

---

## Attack Label Semantics

* `attack_label = 1` is assigned **only when `tx_count > 0`**.
* Coverage is enforced per-node over **active rows only**.
* Attack eligibility is technology-based (configurable).
* Each attack window is documented with:

  * affected nodes
  * time bounds
  * attack strength parameters

See:

```
Dataset/attacks_windows_meta.csv
```

---

## Reproducibility

* Fixed base seed with split-specific offsets.
* Burn-in prevents initialization artifacts.
* All stochastic design choices are logged via configuration and metadata files.

---

## Intended Use

This dataset generator is suitable for:

* centralized machine learning baselines
* topology-aware models (GCN, GAT, spatio-temporal models)
* federated learning experiments using per-client splits

When reporting results, users should clearly state:

* activity-gated attack labeling
* latent channel components not exposed
* train-only normalization
* topology-local passive attack model

---

## License

Choose one and add a `LICENSE` file:

* **MIT License**
* **Apache License 2.0**

---

## Citation

If you use this generator in academic work, please cite the accompanying paper
once available.


