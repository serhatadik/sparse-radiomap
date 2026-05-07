# Transmitter Localization and Radio Map Reconstruction from Sparse Sensor Networks

Code for transmitter localization and radio-map reconstruction from spatially sparse sensor measurements. Two estimation pipelines are implemented end-to-end and evaluated on the POWDER 5-transmitter campaign:

- **Likelihood method** (`src/localization/`): closed-form transmit-power fit per grid cell, Gaussian likelihood with spatial-shadow covariance, marginalized power-field prediction.
- **GLRT-based sparse reconstruction** (`src/sparse_reconstruction/`): pre-whitened observation model, sequential GLRT detection with beam search and physics filters, candidate-pool refinement, IC-based subset selection, optional per-TX path-loss-exponent refit.

Both pipelines plug into three propagation models in `src/propagation/`: log-distance, TIREM, and NVIDIA Sionna RT.

---

## Repository layout

```
latent-sc/
├── config/
│   ├── parameters.yaml             # Pipeline hyperparameters
│   ├── sionna_parameters.yaml      # Ray-tracing settings
│   ├── tirem_parameters.yaml       # TIREM settings
│   ├── transmitter_locations.yaml  # POWDER TX coordinates
│   └── monitoring_locations_legacy.yaml
├── notebooks/
│   ├── likelihood_localization.ipynb
│   ├── glrt_localization.ipynb
│   ├── sparse_reconstruction_localization.ipynb
│   ├── visualize_sionna_scene.ipynb
│   └── paper_reproduction.ipynb
├── scripts/                        # Pipeline drivers, sweeps, IQ ingestion
│   └── sweep/                      # Multi-config experiment orchestration
├── src/
│   ├── localization/               # Likelihood pipeline
│   ├── sparse_reconstruction/      # GLRT pipeline (whitening, glrt_solver, reconstruction)
│   ├── propagation/                # log_distance, tirem_wrapper, sionna_wrapper
│   ├── evaluation/                 # validation, reconstruction_validation, metrics
│   ├── tirem/                      # TIREM bindings
│   ├── data_processing/            # IQ → power, occupancy, temporal
│   ├── interpolation/              # IDW baseline
│   ├── analysis/                   # Correlation, regression
│   ├── utils/                      # Coordinates, units, map I/O
│   └── visualization/
└── tests/
```

---

## Setup

```bash
pip install -r requirements.txt
pip install -e .
```

Sionna ray tracing is optional; install `sionna-rt` and Mitsuba 3 only when using the ray-tracing propagation model.

---

## Running the pipelines

**Likelihood reconstruction:**
```bash
python scripts/run_likelihood_reconstruction.py
```

**Ablation study (multi-config sweep over whitening, beam search, IC selection, physics filters, etc.):**
```bash
python scripts/run_ablation_study.py
```
Plotting of ablation outputs is driven by `scripts/ablation_plotting.py`.

**Hyperparameter and reconstruction sweeps:**
```bash
python scripts/hyperparam_sweep.py
python scripts/reconstruction_parameter_sweep.py
python scripts/comprehensive_parameter_sweep.py
```

**Propagation matrix precomputation (cache once, reuse across runs):**
```bash
python scripts/precompute_sionna_matrices.py
python scripts/precompute_validation_matrices.py
```

**Interactive exploration:**
```bash
jupyter notebook notebooks/glrt_localization.ipynb
jupyter notebook notebooks/likelihood_localization.ipynb
jupyter notebook notebooks/sparse_reconstruction_localization.ipynb
```

The legacy occupancy / signal-estimation pipeline is preserved in `scripts/01_process_occupancy.py` through `scripts/04_generate_figures.py`, with `scripts/run_full_pipeline.py` as a single entry point.

---

## Configuration

Algorithm hyperparameters (path-loss exponent, decorrelation distance, IC weights, beam width, physics-filter thresholds, etc.) live in `config/parameters.yaml`. Per-model settings are split into `config/tirem_parameters.yaml` and `config/sionna_parameters.yaml`. Ground-truth POWDER transmitter coordinates are in `config/transmitter_locations.yaml`.

Propagation matrices are computed on demand and cached, keyed by sensor configuration, grid shape, and model parameters; subsequent runs reuse the cache.

The orchestration framework that pairs sensor placements with experiment configurations and dispatches them across cores lives in `scripts/sweep/`; see `scripts/sweep/README.md` for its usage.

---

## Custom IQ data pipeline

For ingesting raw SDR captures into the monitoring-location format used by the notebooks:

```bash
python scripts/process_raw_data_to_monitoring.py \
    --input-dir <path/to/samples_*> \
    --transmitter <ebc|ustar|guesthouse|mario|moran|wasatch> \
    --num-locations 10 \
    --output-yaml config/monitoring_locations_<tx>.yaml
```

The script recursively finds `samples_*` directories, computes PSDs at the channel of interest, matches with GPS (±10 s), applies the EBC/USTAR June-28 date split, and aggregates within a 20 m radius. A companion script `scripts/process_raw_data_to_validation.py` builds held-out validation sets from independent captures.

| Transmitter   | Channel | Frequency (MHz)       | Date range          |
|---------------|---------|-----------------------|---------------------|
| `ebc`         | TX1     | 3533.904 – 3533.931   | ≤ 2023-06-27        |
| `ustar`       | TX1     | 3533.904 – 3533.931   | ≥ 2023-06-28        |
| `guesthouse`  | TX2     | 3533.945 – 3533.973   | all                 |
| `mario`       | TX3     | 3533.986 – 3534.014   | all                 |
| `moran`       | TX4     | 3534.028 – 3534.055   | all                 |
| `wasatch`     | TX5     | 3534.069 – 3534.096   | all                 |
