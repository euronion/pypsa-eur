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
i.e. fixed afterwards and updated to math the current version on upstream/master.

---
**2019-12-03**

* Model still contains only very low amounts of hydrogen and battery storage used.
* Model was recreated with 45 clusters to increase spatial resolution a little bit
    * It is intresting to see Germany and France split into multiple clusters
    * Also the UK is split into more clusters, with significant transmission going on between England and Scottland

## Model with Stores instead of Storage Units

To see, if the problem with low battery and hydrogen investment is due to a modelling decision
of representing both as Storage Units, redo the simulation

* Define both storage units now as stores
    * Stores are independent
    * Connect to links (transformers) with independend optimisation
    * Now 3 investment and optimisation options for both technologies each
    * Configuration done by changing in `config.yaml`:
        `electricity.extenable_carriers.storage_units` and 
        `electricity.extenable_carriers.stores`
    * Existing simulation results are renamed from `elec_s_45_ec_lcopt_Co2L-3H.nc` to `elec_s_45_ec_lcopt_Co2L-3H_storage-units`
    * Deleting everything in the workflow which depends on the change, i.e. the output of rule `add_extra_components`
    * Then manually executed the rule to recreate the network with attatched extra components `snakemake networks/elec_s_45_ec.nc`
    * Increased number of cores in `Snakefile` for most rules
    * Rerun the solving
    ```
        snakemake --cores=16 --shadow-prefix=/tmp/snakemake-shadow/ results/networks/elec_s_45_ec_lcopt_Co2L-3H.nc
    ```
    Since the `networks/elec_s_45_ec.nc` changed, the depending downstream rules are rerun.

# 2019-12-05

For plotting results, the analysis script was modified slightly:
The legend displaying the bubble size and respective capacity (5, 10 GW) where not displayed correctly.
(Problem: Adding Circle Patches to a matplotlib legend is not directly possible.
Adding a scatter plot as legend does not correctly preserve the sizes, so Patches were added to fake a legend
and represent the bubble sizes correctly.)

Results of the 2019-12-03 run are similar to the run before: Nearly no hydrogen is deployed (~10 MW max. on one node) and PHS / Hydro dam is providing for most of the (long-term) storage. Batteries are only deployed in southern countries.

![](Optimised%20generation%20capacities..png)
![](Optimised%20storage%20capacities.png)
![](State%20of%20charge%20all%20storages.png)

## Modifying hydro capacities

Possible cause could be too high hydro capacities, trying a new run using the option
```
    hydro_max_hours: "estimate_by_large_installations"   # one of energy_capacity_totals_by_country,
```

and introducing a `run` wildcard in the `Snakefile`, to store results with a prefix `results/networks//{run}_xyz.nc`.
To rerun the model, deleting the output of rule `add_electricity` (which is influenced by this config change) and rerunning snamekake again

# 2019-12-06

* Disable downloading of datafiles (`retrieve_databundle`) in `Snakefile` workflow via `config.yaml` -
  the files are already downloaded, so we do not need to redownload them everytime / delete them via the
  rule for deleting outputs.