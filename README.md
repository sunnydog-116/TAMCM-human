# Volumetric Bodyfit

`volumetric_bodyfit` is a volumetric human body fitting toolkit. It loads scan meshes or preprocessed voxel data, predicts displacement fields for body-template vertices, and can run body model fitting, Chamfer refinement, surface-offset refinement, correspondence transfer, and evaluation reports.

This repository contains the refactored source code, lightweight runtime profiles, and small resource files. Large checkpoints, licensed body-model assets, and dataset artifacts are intentionally kept outside this tree and mounted through environment variables.

## Project Layout

```text
.
|-- src/volumetric_bodyfit/
|   |-- config/          # Runtime paths and environment variable handling
|   |-- dataflow/        # Dataset and Lightning DataModule code
|   |-- fieldnets/       # Voxel encoders and point-query networks
|   |-- solver/          # Training system, geometry utilities, and fitting logic
|   |-- entrypoints/     # Batch, CAPE, 4D-DRESS, and interactive entrypoints
|   |-- reports/         # Error reports and correspondence evaluation
|   `-- resources/       # Pair lists and body vertex groups
|-- profiles/            # Inference and evaluation runtime profiles
|-- model_profiles/      # Model configuration profiles, without large checkpoints
|-- artifact_manifest.csv
|-- install.sh
|-- requirement.txt
|-- setup.cfg
`-- surface_asset_export.py
```

## Artifact Layout

Large files are not stored in this refactored source tree. At runtime, the code expects `BODYFIT_ARTIFACT_ROOT` to point to a folder with this structure:

```text
<artifact-root>/
|-- storage/             # Checkpoint folders
`-- support_data/        # Body models and support files
```

`artifact_manifest.csv` lists the large files that were left outside the refactored tree. The artifact names in that manifest are anonymized and are only meant to preserve count and size information.

## Checkpoint

A checkpoint is provided as an external release asset rather than a Git-tracked file, because checkpoint binaries are large and should remain separate from the source tree.

- File name: `checkpoint.ckpt`
- Download: [checkpoint.ckpt](https://github.com/sunnydog-116/TAMCM-human/releases/download/v1.0/checkpoint.ckpt)

After downloading the checkpoint, place it under the artifact root, for example:

```text
<artifact-root>/
`-- storage/
    `-- pretrained_bodyfit/
        `-- checkpoints/
            `-- checkpoint.ckpt
```

Then configure the runtime path with:

```powershell
$env:BODYFIT_ARTIFACT_ROOT = "<path-containing-storage-and-support_data>"
$env:BODYFIT_CHECKPOINT_DIR = "<artifact-root>/storage"
```

## Environment Setup

The project is designed for a reproducible research environment with Python 3.8, CUDA-enabled PyTorch, mesh-processing libraries, and external human body-model assets. A GPU environment is strongly recommended for inference and large-scale evaluation, and is generally required for training.

### 1. Create a Conda Environment

```powershell
conda create -n bodyfit python=3.8.13 -y
conda activate bodyfit
python -m pip install --upgrade pip setuptools wheel
```

Python 3.8 is recommended because several geometry-processing and human-body-model dependencies have stricter compatibility requirements than ordinary scientific Python packages.

### 2. Install the CUDA PyTorch Stack

Install PyTorch with the CUDA runtime that matches the target workstation. For CUDA 11.8:

```powershell
conda install -y pytorch torchvision torchaudio pytorch-cuda=11.8 -c pytorch -c nvidia
```

For a different CUDA version, install the matching PyTorch build and keep the same version alignment when installing PyTorch3D.

### 3. Install PyTorch3D

PyTorch3D should be installed separately because its binary compatibility depends on the active Python, PyTorch, CUDA, and compiler versions. Use a prebuilt wheel when available, or build it from source inside the same Conda environment.

Verify the installation:

```powershell
python -c "import torch; import pytorch3d; print(torch.__version__); print(torch.cuda.is_available())"
```

### 4. Install Project Dependencies

Install the pinned Python dependencies from the repository root:

```powershell
cd "<path-to-this-project>"
pip install -r requirement.txt
```

The final line of `requirement.txt` installs the local package in editable mode with `pip install -e .`, which is convenient for research workflows where source files may be modified during experimentation.

### 5. Install External Research Dependencies

The following components are intentionally not vendored in this repository and should be installed from approved source trees, institutional mirrors, or official releases:

- `human_body_prior` or an interface-compatible body-prior package
- voxel preprocessing extensions used to transform meshes into volumetric tensors
- body-model fitting utilities compatible with the runtime interfaces
- licensed SMPL/SMPL-X or related body-model files, if required by the selected profile

These dependencies often have independent academic or dataset-specific licenses. Keep them outside the Git repository and expose them through environment variables.

### 6. Configure Runtime Paths

At minimum, define the project root and artifact root:

```powershell
$env:BODYFIT_PROJECT_HOME = "<path-to-this-project>"
$env:BODYFIT_ARTIFACT_ROOT = "<path-containing-storage-and-support_data>"
```

The artifact root should contain:

```text
<artifact-root>/
|-- storage/
|   `-- <checkpoint-name>/checkpoints/
`-- support_data/
    `-- <body-model-and-support-files>
```

Common optional overrides:

```powershell
$env:BODYFIT_OUTPUT_DIR = "<output-folder>"
$env:BODYFIT_CHECKPOINT_DIR = "<checkpoint-folder>"
$env:BODYFIT_MODEL_PROFILE_DIR = "<model-profile-folder>"
$env:BODYFIT_PROCESSED_DATA_DIR = "<processed-training-data>"
$env:BODYFIT_DEMO_SCAN_DIR = "<demo-scan-folder>"
$env:BODYFIT_ROTATED_DEMO_SCAN_DIR = "<rotated-demo-scan-folder>"
```

Dataset-specific optional overrides:

```powershell
$env:BODYFIT_CAPE_EVAL_DIR = "<cape-eval-folder>"
$env:BODYFIT_CAPE_RAW_DIR = "<cape-raw-folder>"
$env:BODYFIT_CAPE_SMPL_DIR = "<cape-body-model-info>"
$env:BODYFIT_DRESS_EVAL_DIR = "<dress-eval-folder>"
$env:BODYFIT_DRESS_SMPL_DIR = "<dress-body-model-info>"
$env:BODYFIT_FAUST_SCAN_DIR = "<faust-scan-folder>"
$env:BODYFIT_FAUST_REGISTRATION_DIR = "<faust-registration-folder>"
```

### 7. Validate the Installation

Run a syntax check from the repository root:

```powershell
python -m compileall -q .
```

Verify that the package imports:

```powershell
python -c "import volumetric_bodyfit; print(volumetric_bodyfit.__name__)"
```

For a GPU-capable environment, also check CUDA availability:

```powershell
python -c "import torch; print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'CPU only')"
```

If import errors occur, first check PyTorch/PyTorch3D compatibility, then confirm that external body-model and voxel-processing packages are installed in the active Conda environment.

## Main Entrypoints

Batch fitting:

```powershell
python -m volumetric_bodyfit.entrypoints.batch_register
```

CAPE workflows:

```powershell
python -m volumetric_bodyfit.entrypoints.cape_reconstruction
python -m volumetric_bodyfit.entrypoints.cape_scan_reconstruction
```

4D-DRESS workflow:

```powershell
python -m volumetric_bodyfit.entrypoints.dress_reconstruction
```

Interactive demo:

```powershell
streamlit run src/volumetric_bodyfit/entrypoints/interactive_app.py
```

Evaluation reports:

```powershell
python -m volumetric_bodyfit.reports.faust_report
python -m volumetric_bodyfit.reports.cape_report
python -m volumetric_bodyfit.reports.dress_report
python -m volumetric_bodyfit.reports.pair_transfer
```

## Configuration

Runtime profiles live in `profiles/`. Model profiles live in `model_profiles/`.

Hydra targets point to the refactored package structure:

```text
volumetric_bodyfit.dataflow.lightning_bridge.ShapeFieldDataModule
volumetric_bodyfit.dataflow.surface_samples.SmplFieldDataset
volumetric_bodyfit.solver.system.FieldRegistrationModule
```

To switch models, update `core.checkpoint` in one of the `profiles/*.yaml` files. The matching model configuration should exist at:

```text
model_profiles/<checkpoint-name>/config.yaml
```

The matching checkpoint archive should exist at:

```text
<checkpoint-folder>/<checkpoint-name>/checkpoints/*.zip
```

## Data Preparation

Training data is expected under:

```text
<processed-data>/<version>/stage_III/<split>/ifnet_indi/
```

Typical split names are `train`, `val`, and `test`. Each sample should include both vertex `.pt` files and voxel `.pt` files.

Inference inputs are selected by `profiles/*.yaml` and the environment variables above. Before running a full batch, test with a small number of `.ply` or `.obj` meshes to confirm paths, scale handling, and output folders.

## Outputs

The default output folder is:

```text
<project-home>/output/
```

Override it with `BODYFIT_OUTPUT_DIR` when needed. Common outputs include:

- aligned input meshes
- body-template fitting results
- Chamfer-refined results
- surface-offset refined results
- error logs and visualization meshes

## Development Checks

Syntax check:

```powershell
python -m compileall -q .
```

The refactored tree should not contain old package names, old file names, personal names, email addresses, or personal absolute paths.
