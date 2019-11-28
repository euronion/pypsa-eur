# Notes for 'Hambach' scenario modelling

---
**2019-11-20**


* Config created and based on PyPSA master branch (2019-11-19)
* Config modified
    * for 37 clusters (suggested minimum in `config.default.yaml`)
    * Only do one type of scenario simulation per command invocation
* Preliminary target
  Only do one scenario simulation to test the new structure and analysis scripts from the upstream PyPSA/pypsa-eur repository.

## Environment (re-)installation

Due to problems with powerplant matching (requires pyyaml > 5.1 for `FullLoader` but standard environment automatically installs 3.X version), the environment was reinstalled:

* Modified the `environment.yaml` with additional packages and version restrictions
* Deleted the environment

```
conda deactivate
conda env remove --name pypsa-eur
```

* Reinstalled the environment from file

```
conda env create -f environment.yaml
```

* Activate the environment:
 
```
conda activate pypsa-eur
```

* Exported the installed package versions

```
conda env export --no-builds -f environment.installedversions.yaml
```

## Solving the network

Triggering building and solving the network based on `config.yaml` with

```
snakemake --shadow-prefix=/tmp/snakemake-shadow/ solve_all_elec_networks
```

and for summaries
```
snakemake results/summaries/elec_s_37_lcopt_Co2L-3H_all
```

Both commands run without any problems in ~3-4 hours (tops).

---
**2019-11-21**

## Changes

* Changed "hydro"="Hydro" label to "Hydro dams" to make the distinction in technologies clear
    * Hydro (dams): Are similar to ROR, but have a significant reservoir making them a dispatchable energy source
    * ROR are run-of-river power plants with no or no significant reservoir for storing water, thus making them dependent on
        percipitation and a variable renewable energy source
    * PHS are pumped hydro storage facilities which are dispatachable but not self-regenerating, require power input for charging
* Changed color for "hydro" (Hydro dams) to "#4682b4"; before PHS and hydro had the same color and could not be distinguished
* Modified analysis script to plot all 4 plots in one call

## Analysis

* Most notable and suprising:
    * Low amount of onshore wind in all of Europe: Only in Denmark and Poland notable capacities
    * Low amount of battery storage: Only in southern European countries
    * No Hydrogen H2 storage capacities build
    * Storage is, except for in Spain/Sardinia/Greece where battery storage is expanded, solely done by Hydro dams and PHS

## Model updating to newer version

Before continuing in a more in-depth analysis, exclude possibility for incompatible changes:

* The repository and model used is nearly up-to-date, but not fully up-to-date with the GitHub Repository
* The data bundles used are not updated automatically from GitHub

The original update to the new version did not change the databundles, as they are `.gitignore`d.

Steps for updating to current version

* Archive the current state: as a tarball, indepdendent from GitHub (with time-stamp)

```
    tar -cvf pypsa-eur_hambach_20191121_1205.tar pypsa-eur/
```

* update the current master branch and working branch 'hambach' to current upstream/master repository
    (via git)
* delete the following existing folders which are part of the data bundle or created during the snakemake workflow:

```
    rm -r cutouts
    rm resources/*
    rm -r data/bundle
    rm -r test
    rm -r networks
    rm -r results
    rm -r logs
    rm -r snakemake
```

* recreate / download the data bundles as currently suggested by the tutorial and workflow

```
    snakemake retrieve_databundle
    curl -OL "https://zenodo.org/record/3517949/files/pypsa-eur-cutouts.tar.xz"
    curl -L "https://zenodo.org/record/3518215/files/natura.tiff" -o "resources/natura.tiff"
    tar xJf pypsa-eur-cutouts.tar.xz
    rm pypsa-eur-cutouts.tar.xz
```

## Running the model

Syntax changed slightly with the update, additional 'ec' necessary (also added in `Snakefile` where it was missing from one rule).
To solve the network:

```
    snakemake --cores=10 --shadow-prefix=/tmp/snakemake-shadow/ results/networks/elec_s_37_ec_lcopt_Co2L-3H.nc
```

The problem of the missing 'ec' from the `Snakefile` was due to an incorrect manual merge on my side,
i.e. fixed afterwards and updated to math the current version on upstream/master