# diffRL: Analyzing Symbolic Properties for DRL Agents in Systems and Networking

This repository contains the source code and experimental artifacts for **diffRL**, a framework for formally verifying safety and performance properties of deep reinforcement learning (DRL) policies deployed in systems applications. diffRL bridges the gap between DRL-based controllers used in real-world networked systems and formal verification engines by performing *comparative encoding* and *property decomposition* — translating high-level system-specific properties into solver-compatible symbolic sub-queries that can be dispatched to multiple verification backends.

<p align="center">
  <img src="diffRL.png" alt="diffRL pipeline overview" width="700"/>
</p>

The pipeline works as follows: given a trained DNN policy and a property specification (input ranges, slack, tolerances), diffRL's **comparative encoding** reformulates the property into a form amenable to neural network verifiers, and the **decomposition** step breaks it into individual sub-properties (queries) in VNN-LIB format. These queries are then solved by one or more verification engines — a BaB-based engine (alpha-beta-CROWN), a MIP solver (via Gurobi inside alpha-beta-CROWN), or an SMT-based engine (Marabou). Results are aggregated to produce the final verification outcome.

## Applications

diffRL is evaluated on four DRL-based systems applications:

| Application | Domain | Description |
|---|---|---|
| **Pensieve** | Adaptive bitrate streaming | Selects video bitrate to maximize quality of experience |
| **Aurora** | Congestion control | Learns sending rates to optimize network throughput |
| **CMARS** | RAN slicing | Schedules resources across RAN slices |
| **FIRM** | Serverless computing | Manages function invocations to meet SLO targets |

For each application, diffRL verifies properties such as **robustness**, **capacity utilization**, **loss/rebuffering avoidance**, **multiplexing gain**, **channel compensation**, and **SLO preservation**.

## Repository Structure

```
sys-rl-verif/
├── applications/                  # Benchmark data for each application
│   ├── aurora/                    # Aurora congestion control
│   │   ├── aurora_lib/            #   Application-specific RL code and environment
│   │   ├── models/                #   Trained DNN policy models (ONNX)
│   │   ├── vnnlib/                #   VNN-LIB property files organized by property type
│   │   ├── *.csv                  #   Property CSV manifests listing vnnlib paths
│   │   └── preparation.ipynb      #   Notebook for model export and property generation
│   ├── cmars/                     # CMARS multi-access scheduling
│   │   ├── cmars_lib/             #   Application-specific RL code
│   │   ├── models/                #   Trained DNN policy models (ONNX)
│   │   ├── vnnlib/                #   VNN-LIB properties (output_15/ and output_30/)
│   │   ├── *.csv                  #   Property CSV manifests
│   │   └── preparation.ipynb      #   Notebook for model export and property generation
│   ├── firm/                      # FIRM serverless computing
│   │   ├── firm_lib/              #   Application-specific RL code and workflow data
│   │   ├── model/                 #   Trained DNN policy models
│   │   ├── vnnlib/                #   VNN-LIB properties (slo_preservation/, robustness/)
│   │   ├── *.csv                  #   Property CSV manifests
│   │   └── preparation.ipynb      #   Notebook for model export and property generation
│   └── pensieve/                  # Pensieve adaptive bitrate streaming
│       ├── pensieve_lib/          #   Application-specific RL code
│       ├── model/                 #   Trained DNN policy models
│       ├── vnnlib/                #   VNN-LIB properties
│       ├── *.csv                  #   Property CSV manifests
│       └── preparation.ipynb      #   Notebook for model export and property generation
├── alpha-beta-CROWN/              # BaB and MIP-based verification (git submodule)
│   ├── complete_verifier/
│   │   ├── abcrown.py             #   Main verifier entry point
│   │   ├── exp_configs/
│   │   │   └── sys-rl-verif/      #   Experiment configs for this project
│   │   │       ├── aurora/        #     mip.yaml, bab_inputsplitting.yaml
│   │   │       ├── cmars/         #     mip.yaml, bab_inputsplitting.yaml
│   │   │       └── pensieve/      #     mip.yaml, bab_inputsplitting.yaml
│   │   ├── aurora_lib/            #   Application model loaders for the verifier
│   │   ├── cmars_lib/
│   │   ├── pensieve_lib/
│   │   ├── firm_lib/
│   │   └── run.sh                 #   Example run script
│   └── auto_LiRPA/                #   Automatic linear relaxation based perturbation analysis
├── marabou/                       # SMT-based verification via Marabou
│   ├── aurora_query.py            #   Run Marabou queries for Aurora
│   └── cmars_query.py             #   Run Marabou queries for CMARS
└── diffrl.pdf                     # Pipeline overview figure
```

Each application directory under `applications/` follows a consistent structure:
- **`*_lib/`** — RL environment, model definitions, and utility code.
- **`models/`** — Trained DNN policies exported to ONNX format.
- **`vnnlib/`** — VNN-LIB specification files encoding the verification properties.
- **`*.csv`** — Manifest files listing the `(model_path, vnnlib_path)` pairs for batch verification.
- **`preparation.ipynb`** — Jupyter notebook that exports trained models and generates VNN-LIB properties.

## Requirements

### 1. Conda Environment

The alpha-beta-CROWN verifier requires a conda environment. Set it up from the repository root:

```bash
conda env create -f alpha-beta-CROWN/complete_verifier/environment.yaml --name alpha-beta-crown
conda activate alpha-beta-crown
```

### 2. Gurobi License

A [Gurobi](https://www.gurobi.com/) license is required for **MIP-based verification**. Install a license using the `grbgetkey` command. If you do not have access to a full license, the conda installation above includes a free restricted license that is sufficient for the relatively small networks used in this project. A Gurobi license is *not* needed if you only use BaB-based verification (alpha-CROWN / beta-CROWN).

### 3. GPU (for BaB-based verification)

BaB-based verification with alpha-beta-CROWN benefits significantly from GPU acceleration. A CUDA-capable GPU is recommended when running the `bab_inputsplitting` configurations. MIP-based verification runs on CPU.

### 4. Marabou

To run experiments with the Marabou SMT-based verifier, follow the [Marabou installation instructions](https://github.com/NeuralNetworkVerification/Marabou). The Python interface `maraboupy` must be available in your Python environment.

## Running Experiments

### Alpha-Beta-CROWN (BaB and MIP)

All alpha-beta-CROWN experiments are launched from `alpha-beta-CROWN/complete_verifier/` using the `abcrown.py` entry point with a YAML configuration file.

#### MIP-based Verification

MIP-based verification encodes the neural network and the property as a Mixed-Integer Program and solves it with Gurobi. Runs on CPU.

```bash
cd alpha-beta-CROWN/complete_verifier
conda activate alpha-beta-crown

# Aurora
python abcrown.py --config exp_configs/sys-rl-verif/aurora/mip.yaml

# CMARS
python abcrown.py --config exp_configs/sys-rl-verif/cmars/mip.yaml

# Pensieve
python abcrown.py --config exp_configs/sys-rl-verif/pensieve/mip.yaml
```

Each config specifies the application model, input shape, property CSV manifest, and solver parameters (timeout, thread count, etc.). You can modify the `csv_name` and `model.name` fields in the YAML files to run different property sets or model variants.

#### BaB-based Verification (Branch and Bound with Input Splitting)

BaB-based verification uses bound propagation (alpha-CROWN, beta-CROWN) with branch-and-bound and input splitting. Benefits from GPU.

```bash
cd alpha-beta-CROWN/complete_verifier
conda activate alpha-beta-crown

# Aurora
python abcrown.py --config exp_configs/sys-rl-verif/aurora/bab_inputsplitting.yaml

# CMARS
python abcrown.py --config exp_configs/sys-rl-verif/cmars/bab_inputsplitting.yaml

# Pensieve
python abcrown.py --config exp_configs/sys-rl-verif/pensieve/bab_inputsplitting.yaml
```

### Marabou (SMT-based Verification)

Marabou experiments are run from the `marabou/` directory. Each script reads an ONNX model and iterates over properties listed in a CSV manifest.

```bash
cd marabou

# CMARS
python cmars_query.py

# Aurora
python aurora_query.py
```