# aeon_swc_rigs

Core Pydantic schemas for describing Aeon rigs, Harp devices, and experiment metadata.

This package is designed to be imported and used as core infrastructure by experiment-specific repositories (for example,
[`aeon_exp_foragingABC`](https://github.com/SainsburyWellcomeCentre/aeon_exp_foragingABC)).

By convention these repos should:

- be named with the prefix `aeon_exp_*`, or `aeon_qc_*` for testing and quality control rigs respectively
- define their own rig and task schemas by subclassing the models in `swc.aeon_rigs`
- generate JSON Schema from those models using `pydantic`
- use the JSON schemas to generate Bonsai extensions using `Bonsai.sgen` and to validate YAML configuration files.

The environment (defined by a `pyproject.toml`) and pydantic models defining classes should always be version controlled. Automatically generated JSON and CSharp classes can also be version controlled to avoid python, dotnet and bonsai.sgen dependencies to deploy to a new rig. Alternatively, or additionally, these can be automatically generated in a `deploy.ps1` script.

## Using `swc-aeon-rigs` in a new experiment repository

### 1. Name and version your project, and declare `swc-aeon-rigs` as a dependency. 

This should be done in a new file named `pyproject.toml` and add the `.git` repository as a source. Example below:

```toml
[project]
name = "aeon_exp_myTask"
version = "0.1.0"
requires-python = ">=3.11"
dependencies= [
  "swc-aeon-rigs"
]

[tool.ruff]
line-length = 108

[tool.uv.sources]
swc-aeon-rigs = { git = "https://github.com/SainsburyWellcomeCentre/aeon_swc_rigs.git" }
```

## 2. Install the uv python package manager

We recommend uv for easy and fast python environment and package management

Install Python if not already installed: https://www.python.org/
Install `uv` if not already installed:

On Windows, to get the latest version:
```
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```
For further information or other operating systems, see the [documentation pages](https://docs.astral.sh/uv/getting-started/installation/) for `uv` 

## 3. Create python environment

```bash
uv sync
```

This installs:
- your experiment package
- swc-aeon-rigs
- all other dependencies listed in the `pyproject.toml`.

### 3. Create your experiment package structure

```
src/
  swc/
    aeon_exp/
      myTask/
        rig.py
        task.py
        experiment.py
```

`rig.py` composes and expands on devices from `swc.aeon_rigs.*` to describe this rig.  

`task.py` comprises all your task parameters.

`experiment.py` subclasses `swc.aeon_rigs.experiment.Experiment` and describes your experiment metadata.

### 4. Generate the JSON Schema

```bash
uv run python -m swc.aeon_exp.myTask.experiment
```

This writes `MyExperiment.json` (or whatever filename your `main()` function within `experiment.py` outputs).

### 5. Generate Bonsai extensions

Install or restore Bonsai’s dotnet tools. You must have dotnet installed.
One way is to copy the `dotnet-tools.json` in the `.config` directory of this repository to your experimental repo, then run:

```bash
dotnet tool restore
```
See [Bonsai.Sgen](https://bonsai-rx.org/sgen/) documentation for more details.
Generate typed extension nodes with `Bonsai.Sgen`:

```bash
dotnet bonsai.sgen MyExperiment.json -o Extensions --serializer yaml
```

This produces `*.Generated.cs` C# definitions under the `Extensions/` directory, which Bonsai will then load automatically when run from this environment.
