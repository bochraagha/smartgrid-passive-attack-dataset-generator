# SmartGrid Passive Attack Benchmarking Dataset Generator (HAN/NAN/WAN)

Python script to generate a synthetic smart-grid communication dataset with HAN/NAN/WAN nodes, an IEEE-ish adjacency graph, complex Gauss–Markov (AR(1)) fading, literature-anchored shadowing/interference processes, and presence-only passive attack windows (shadow-loss + coherence drop). Outputs per-node (client) CSV files for train/val/test together with metadata, adjacency/weights, attack-window logs, and saved per-node z-score statistics.

---

## Requirements

- Python 3
- `numpy`
- `pandas`

Optional (Colab only): `google.colab` for Drive mount.

---

## Configuration

Edit the `Config` dataclass:

- **Output**
  - `OUT_DIR` (default: `/content/drive/MyDrive/Complex`)
  - `OVERWRITE` (delete and recreate `OUT_DIR` if exists)

- **Reproducibility**
  - `SEED_BASE` (split seeds derived from this)

- **Split lengths (per node)**
  - `T_TRAIN`, `T_VAL`, `T_TEST`
  - `BURN_IN` (not exported; stabilizes AR/Markov processes)
  - `DT_SECONDS`

- **Attack placement**
  - `TARGET_ATTACK_FRAC` (target fraction over ACTIVE rows on eligible nodes)
  - Windowing: `WIN_CORE_MIN`, `WIN_CORE_MAX`, `LEAD`, `TAIL`, `HYST`
  - Group size: `GROUP_MIN`, `GROUP_MAX`
  - `ALLOW_OVERLAP`, `UNIQUE_KEY`
  - Activity constraints: `MIN_ACTIVE_FRAC_CORE`, `MIN_ACTIVE_FRAC_LABELED`, `ACTIVITY_RESAMPLE_TRIES`
  - Eligible tech: `ATTACK_ELIGIBLE_TECH`

- **Presence-only attack parameters**
  - Shadow-loss mixture: `ATTACK_SHADOW_*`
  - Coherence drop mixture: `ATTACK_KDROP_*`, mapping to surrogates via `ATTACK_ALPHA_*` and `ATTACK_SIGMA_*`
  - Markov bad-state: `ATTACK_GE_*`, `ATTACK_BAD_*`
  - Ramp: `RAMP_FRAC`
  - WiFi reflection (optional): `WIFI_*`

---

## Nodes

Nodes are defined in `NODES` as `(node_id, role, layer, tech)`:

- HAN: SmartMeter (ZigBee/WiFi), Gateway (WiFi)
- NAN: DER (LoRa), FeederRelay (PLC), Controller (LTE)
- WAN: PMU (Fiber), SCADA (Fiber), AMIHeadend (LTE), SubstationGW (PLC)

---

## How to run

Run the script end-to-end (Colab or local). The last lines execute generation:

```python
df_splits, df_meta = main(return_dfs=True)
print("OUT_DIR:", cfg.OUT_DIR)
```

If using Colab + Drive, the script attempts to mount `/content/drive` automatically.

---

## Output structure

`OUT_DIR/` contains:

- `config.json` (full configuration)
- `reason_vocab.json`
- `metadata.csv` (node metadata)
- `metadata_ohe.csv` (one-hot role/layer/tech)
- `network_adjacency.csv` (adjacency matrix `A`)
- `network_weights_W.csv` (row-stochastic mixing matrix `W`)
- `attacks_windows_meta.csv` (attack windows + sampled parameters)
- `clients/`
  - `client_node{0..11}/`
    - `train.csv`
    - `val.csv`
    - `test.csv`
    - `train_stats.json` (per-node z-score statistics)

---

## What each per-client CSV contains

Each row corresponds to one `(t, node)`:

- **Indexing/labels**: `t`, `node`, `split`, `attack_label`, `tx_count`
- **Channel/telemetry**: `csi_amp`, `snr_db`, `packet_error`, `latency`, `latency_smooth`, `shadow_db`, `interf_db`
- **Phase (from complex channel)**: `phase_rad`, `phase_sin`, `phase_cos`, `dphase`
- **Rolling/derived**: `csi_entropy`, `csi_drift`, `csi_skewness`, `csi_kurtosis`, `time_since_last_tx`
- **Neighbor features** (via `W`): `avg_neighbor_*`, `rho_like_*`
- **Sensitivity features**: `csi_db`, `dcsi_db`, `dsnr_db`, `snr_minus_csi_db`, `per_logit`, `dper_logit`, `lat_log`, `dlat_log`,
  plus rolling autocorrelation/innovation and robust/MAD and quantile-spread features
- **Z-scores**: `z_*` for the configured feature list (`z_cols`)
- **CUSUM/EWMA**: `cusum_*` and `ewma_*` for selected z-score columns, and graph delta/energy terms when available

---

## Z-score fitting (no leakage)

Per-node z-score statistics are fitted using:

- Split: **train only**
- Rows: **`attack_label == 0`** and **`tx_count > 0`**

Saved to:

- `clients/client_node{node}/train_stats.json`

Applied to train/val/test using those saved stats.

---

## Attack labeling

- Attack windows are placed to target `TARGET_ATTACK_FRAC` over **ACTIVE** rows (`tx_count > 0`) on eligible technologies (`ATTACK_ELIGIBLE_TECH`).
- Windows are created for connected node groups (size `GROUP_MIN..GROUP_MAX`) using the adjacency graph.
- Attack window metadata and sampled parameters are stored in `attacks_windows_meta.csv`.
