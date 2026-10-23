# Pixi Environment Setup Guide

This guide sets up the ParaValve environment with [Pixi](https://pixi.sh). Pixi reads the provided `pixi.toml` manifest, installs the conda-forge and GitHub dependencies, and records the exact versions in `pixi.lock`. You do not need to create or activate an environment by hand.

This is an alternative to the [Conda Environment Setup Guide](Conda_Installation.md) and the [Docker Installation Guide](Docker_Installation.md).

> **Note for Windows users:**
> Pixi runs on Linux, and on Windows only inside WSL. Install WSL with an Ubuntu distribution and follow this guide from the WSL terminal. The environment targets `linux-64` only.

---

## Prerequisites

- **Linux or WSL (Ubuntu recommended)** on an x86-64 machine.
- **Git**, to clone the repository. Pixi also uses git to fetch the ParaValve dependencies from GitHub.
- **Internet access** to `conda.anaconda.org` (conda-forge) and `github.com` during installation.
- **Disk space:** several GB for the environment, plus space for the meshes and results.
- **Memory:** the weak forms are JIT-compiled on first use and need a lot of RAM, particularly the valve forms. See step 6 for how to avoid running out of memory.

---

## Step-by-Step Instructions

### **1. Install Pixi**

```bash
curl -fsSL https://pixi.sh/install.sh | sh
```

The installer adds Pixi to your `PATH` through `~/.bashrc`. Open a new terminal, or run `source ~/.bashrc`, then check the installation:

```bash
pixi --version
```

### **2. Clone the Repository**

```bash
git clone https://github.com/idealab-isu/ParaValve.git
cd ParaValve
```

Stay in the repository root, the folder that contains `pixi.toml`.

### **3. Install the Environment**

```bash
pixi install
```

This creates the environment in `.pixi/` inside the repository. The first install downloads the packages and builds the GitHub dependencies, so it can take a while. Later installs reuse the cache.

To keep the exact versions in `pixi.lock`, do not run `pixi update` unless you intend to change them.

### **4. Verify the Environment**

```bash
pixi run check
```

A successful run prints `ParaValve environment OK`. The check imports `dolfin` (FEniCS 2019.1), `VarMINT`, `SalaMANdER`, `CouDALFISh`, `tIGAr`, and `ShNAPr`.

You can also see the resolved package versions:

```bash
pixi list
```

The key packages are Python 3.8, `fenics-dolfin` 2019.1.0, `mpich`, `petsc4py`, `numpy`, and `scipy` 1.5.

### **5. Generate the Interior Facets and Stent Intersections**

`Preprocessing.py` must be run once per mesh before the simulation. It runs on a single core, and it takes about a minute for the 16 mm mesh.

The script's default paths are relative to the working directory, so run it from the `ParaValve/` subfolder that contains `ParaValve.py`:

```bash
cd ParaValve
pixi run python Preprocessing.py \
    --mesh-folder ./aorta/mesh-16mm/ \
    --valve-folder ./valves/BHV-16mm/ \
    --valve-smesh-suffix .txt
```

Pixi finds `pixi.toml` in the parent directory, so you do not need to return to the root. The repository also defines this as a task that runs from the root:

```bash
pixi run preprocess
```

> **Warning:** Preprocessing writes its output files into the mesh folder (`aorta/mesh-16mm/` here), and it overwrites any existing files with the same names.

### **6. Run the Simulation**

Run the solver from the `ParaValve/` subfolder.

> **Run in serial first.** The first run JIT-compiles the weak forms, which is slow and needs a lot of memory, especially for the valve. In a parallel run every MPI rank compiles the same forms at the same time, so the memory demand is multiplied by the number of ranks and the run can be killed when the machine runs out of memory. Running once in a single process compiles the forms one time and caches them (in `~/.cache/dijitso`), so later parallel runs can reuse them instead of recompiling. Run once in serial, then use `mpirun`:

```bash
cd ParaValve
pixi run python ParaValve.py \
    --mesh-folder ./aorta/mesh-16mm/ \
    --valve-folder ./valves/BHV-16mm/ \
    --valve-smesh-suffix .txt \
    --num-steps 2
```

After that, run in parallel:

```bash
pixi run mpirun -n 2 python ParaValve.py \
    --mesh-folder ./aorta/mesh-16mm/ \
    --valve-folder ./valves/BHV-16mm/ \
    --valve-smesh-suffix .txt \
    --num-steps 2
```

`--num-steps 2` is a short test run. Remove it, or set `--num-steps` to the value you need, for a real simulation. The defaults are 11201 steps of `1e-4` s, starting at `0.6` s. Results go to `./results/` and restarts to `./restarts/`, both relative to the working directory.

---

## Command-Line Defaults to Override

Several defaults in the scripts do not match the files shipped in the repository, so pass these options explicitly:

| Option | Default | What to use |
|---|---|---|
| `--mesh-folder` | `./aorta/` | `./aorta/mesh-16mm/` or `./aorta/mesh-23mm/` (the meshes are in these subfolders) |
| `--valve-folder` | `./valves/BHV-23mm-finestent/` | `./valves/BHV-16mm/` or `./valves/BHV-23mm/` (the default folder is not in the repository) |
| `--valve-smesh-suffix` | `.dat` | `.txt` (the valve files are named `smesh.1.txt` to `smesh.10.txt`) |

Use the valve folder that matches the mesh size (`16mm` with `BHV-16mm`, and `23mm` with `BHV-23mm`).

---

## Using the Environment

- Run one command in the environment with `pixi run <command>`.
- Open an interactive shell in the environment with `pixi shell`, then type `exit` to leave it.
- Run `pixi info` to see where the environment is installed.

---

## What Is and Is Not Included

- **Included:** all packages `ParaValve.py` and `Preprocessing.py` import, plus `mpich` for `mpirun`. The GitHub dependencies (tIGAr, VarMINT with its SalaMANdER module, CouDALFISh, ShNAPr, and the TSFC/FInAT/COFFEE form compiler) are pinned to the same commits or tags as the Dockerfile.
- **Not included, install separately if needed:**
  - **ParaView** for post-processing. `environment.yml` pins ParaView 5.10.1, but the Pixi environment does not include it. Install ParaView from [paraview.org](https://www.paraview.org/) or add it with `pixi add paraview`.
  - **Gmsh** for regenerating meshes. The meshes are already in the repository.
  - **igakit**, which is only needed for the NURBS module of tIGAr. `ParaValve.py` and `Preprocessing.py` do not import it. The Dockerfile installs it, but installing it under Pixi did not work in testing (its build fails under pip's default build isolation, because it uses the deprecated `numpy.distutils`).

---

## Troubleshooting

- **`pixi: command not found`:** open a new terminal, or run `source ~/.bashrc`.
- **Install fails while fetching a GitHub dependency:** check that `github.com` is reachable from your network, then run `pixi install` again.
- **`WARNING! There are options you set that were not used!` at the start of a run:** PETSc prints this for the script's command-line options. It is harmless.
- **`Calling FFC just-in-time (JIT) compiler` appears for a long time:** this is the expected first-run compilation of the valve forms. Wait for it to finish, and check that the machine has enough memory.
- **`BAD TERMINATION ... EXIT CODE: 15` during compilation:** the machine most likely ran out of memory while several ranks compiled at once. Run once in serial first (step 6). On WSL you can also raise the memory limit in `.wslconfig`. You can confirm an out-of-memory kill with `journalctl -k | grep -i "out of memory"`.
- **`Missing data files for valve geometry.`:** the `--valve-folder` or `--valve-smesh-suffix` option does not match the files in the repository. See the table above.

---

## Additional Notes

- Commit `pixi.lock` with any change to `pixi.toml`, so every user gets the same versions.
- For a smoother experience, consider using Visual Studio Code with the [WSL extension](https://code.visualstudio.com/docs/remote/wsl) to develop directly from your WSL environment.
