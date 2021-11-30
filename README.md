
<!-- README.md is generated from README.Rmd. Please edit that file -->

# CytoTalk

<!-- badges: start -->
<!-- badges: end -->

<div align="center">

<img src="docs/cytotalk_diagram.png" height="480px" />

</div>

## Table of Contents

-   [CytoTalk](#cytotalk)
    -   [Table of Contents](#table-of-contents)
    -   [Overview](#overview)
        -   [Background](#background)
    -   [Getting Started](#getting-started)
        -   [Prerequisites](#prerequisites)
        -   [Installation](#installation)
        -   [Preparation](#preparation)
        -   [Running CytoTalk](#running-cytotalk)
    -   [Update Log](#update-log)
    -   [Citing CytoTalk](#citing-cytotalk)
    -   [References](#references)
    -   [Contact](#contact)

## Overview

We have developed the CytoTalk algorithm for *de novo* construction of a
signaling network (union of multiple signaling pathways emanating from
the ligand- receptor pairs) between two cell types using single-cell
transcriptomics data. The algorithm constructs an integrated network of
intracellular and intercellular functional gene interactions. The
signaling network is identified by solving a prize-collecting Steiner
forest (PCSF) problem based on appropriately defined node prize
(i.e. cell-specific gene activity) and edge cost (i.e. functional
interaction between two genes). The objective of the PCSF problem is to
find an optimal subnetwork in the integrated network that includes genes
with high levels of cell-type-specific expression and close connection
to highly active ligand-receptor pairs.

### Background

Signal transduction is the primary mechanism for cell-cell
communication. scRNA- seq technology holds great promise for studying
cell-cell communication at much higher resolution. Signaling pathways
are highly dynamic and cross-talk among them is prevalent. Due to these
two features, simply examining expression levels of ligand and receptor
genes cannot reliably capture the overall activities of signaling
pathways and interactions among them.

## Getting Started

### Prerequisites

CytoTalk requires a Python module to operate correctly. To install the
[`pcst_fast` module](https://github.com/fraenkel-lab/pcst_fast), please
run this command *before* using CytoTalk:

``` console
pip install git+https://github.com/fraenkel-lab/pcst_fast.git
```

CytoTalk outputs a SIF file for use in Cytoscape. Please [install
Cytoscape](https://cytoscape.org/download.html) to view the whole output
network. Additionally, if you want the final ligand-receptor pathways to
render portable SVG files correctly, you’ll have to have Graphviz
installed and the `dot` executable on your PATH. See the [Cytoscape
downloads page](https://graphviz.org/download/) for more information.

### Installation

If you have `devtools` installed, you can use the `install_github`
function directly on this repository:

``` r
devtools::install_github("tanlabcode/CytoTalk")
```

### Preparation

Let’s assume we have a folder called “scRNAseq-data”, filled with
single-cell RNA sequencing (scRNASeq) datasets. Here’s an example
directory structure:

``` console
── scRNAseq-data
   ├─ scRNAseq_BasalCells.csv
   ├─ scRNAseq_BCells.csv
   ├─ scRNAseq_EndothelialCells.csv
   ├─ scRNAseq_Fibroblasts.csv
   ├─ scRNAseq_LuminalEpithelialCells.csv
   ├─ scRNAseq_Macrophages.csv
   └─ scRNAseq_TCells.csv
```

<br />

⚠ **IMPORTANT** ⚠

Notice all of these files have the prefix “scRNAseq\_” and the extension
“.csv”; CytoTalk looks for files matching this pattern, so be sure to
replicate it with your filenames. Let’s try reading in the folder:

``` r
dir_in <- "~/scRNAseq-data"
lst_scrna <- CytoTalk::read_matrix_folder(dir_in)
table(lst_scrna$cell_types)
```

``` console
#>             BasalCells                 BCells       EndothelialCells 
#>                    392                    743                    251 
#>            Fibroblasts LuminalEpithelialCells            Macrophages 
#>                    700                    459                    186 
#>                 TCells 
#>                   1750
```

The outputted names are all the cell types we can choose to run CytoTalk
against. Alternatively, we can use CellPhoneDB-style input:

``` txt
── scRNAseq-data-cpdb
   ├─ sample_counts.txt
   └─ sample_meta.txt
```

There is no specific pattern required for this type of input, as both
filepaths are required for the function:

``` r
fpath_mat <- "~/scRNAseq-data-cpdb/sample_counts.txt"
fpath_meta <- "~/scRNAseq-data-cpdb/sample_meta.txt"
lst_scrna <- CytoTalk::read_matrix_with_meta(fpath_mat, fpath_meta)
table(lst_scrna$cell_types)
```

``` console
#>   Myeloid NKcells_0 NKcells_1    Tcells 
#>         1         5         3         1
```

Finally, you can compose your own input list quite easily, simply have a
matrix of either count or transformed data and a vector detailing the
cell types of each column:

``` r
mat <- matrix(rpois(90, 5), ncol = 3)
cell_types <- c("TypeA", "TypeB", "TypeA")
lst_scrna <- CytoTalk:::new_named_list(mat, cell_types)
table(lst_scrna$cell_types)
```

``` console
#> TypeA TypeB 
#>     2     1
```

### Running CytoTalk

Without further ado, let’s run CytoTalk!

``` r
# read in data folder
dir_in <- "~/scRNAseq-data"
lst_scrna <- CytoTalk::read_matrix_folder(dir_in)

# set required parameters
type_a <- "Fibroblasts"
type_b <- "LuminalEpithelialCells"

# set optional parameters
cutoff_a <- 0.5
cutoff_b <- 0.5

# run CytoTalk process
results <- CytoTalk::run_cytotalk(lst_scrna, type_a, type_b,
                                     cutoff_a, cutoff_b)
```

``` console
#> [1 / 8] (11:15:28) Preprocessing...
#> [2 / 8] (11:16:13) Mutual information matrix...
#> [3 / 8] (11:20:19) Indirect edge-filtered network...
#> [4 / 8] (11:20:37) Integrate network...
#> [5 / 8] (11:21:44) PCSF...
#> [6 / 8] (11:21:56) Determine best signaling network...
#> [7 / 8] (11:21:58) Generate network output...
#> [8 / 8] (11:21:59) Analyze pathways...
```

All we need for a default run (recommended settings) are the selected
cell types “Macrophages” and “LuminalEpithelialCells”. The most
important optional parameters to look at are `cutoff_a`, `cutoff_b`, and
`beta_max`; details on these can be found in the help page for the
`run_cytotalk` function. As the process runs, we see messages print to
the console for each sub process.

Here is what the resulting output looks like:

``` r
summary(results)
```

``` console
#>                Length Class  Mode   
#> pem            163387 -none- numeric
#> integrated_net      2 -none- list   
#> pcst                3 -none- list   
#> pathways            3 -none- list
```

In the order of increasing effort, let’s take a look at some of the
results. Let’s begin with the “pathways” item. This folder contains
source GV files, which the `dot` executable has also transformed into
PNG images and SVG vector graphics (also includes hyperlink support).
Here is an example pathway neighborhood:

<div align="center">

<img src="docs/pathway.svg" height="480px" />

</div>

Note that the SVG files are interactive, with hyperlinks to GeneCards
and WikiPI. Green edges are directed from ligand to receptor. This is a
subset of the overall network, so how do we view the whole thing?

We have the “cytoscape” folder, which includes a SIF file read to import
and two tables that can be attached to the network and used for styling.
Here’s an example of a styled Cytoscape network:

<div align="center">

<img src="docs/cytoscape_network.svg" height="480px" />

</div>

<br />

There are a number of details we can glean from these graphs, such as
node prize (side of each node), edge cost (inverse edge width),
Preferential Expression Measure (intensity of each color), cell type
(based on color, and shape in the Cytoscape output), and interaction
type (dashed lines for crosstalk, solid for intracellular).

If we want to be more formal with the pathway analysis, we can look at
some scores for each neighborhood in the “pathways” folder. This folder
provides extracted subnetworks, based on the “PCSF\_Network.txt” file.
Additionally, the “PathwayScores.txt” file in the “analysis” folder
contains a summary of the neighborhood size for each pathway, along with
empirical test values that are found by taking random subnetworks from
the “IntegratedNetwork.cfg” configuration file, which is used as input
to the PCSF algorithm. p-Values for node prizes (want to maximize) and
edge costs (want to minimize) are calculated separately.

Finally, we can take a look at some of the textual output. Most of the
text files found in the output folder are used for intermediate
calculations, but they are provided in case you want to check our work.
The final network (includes edges and node attributes) generated by the
prize-collecting Steiner tree (PCST) algorithm can be found in the
“PCSF\_Network.txt” file. If you would like to see the inputs to the
PCST algorithm, check out the “PCSF\_Network.txt” file, but note that
this does not include the artificial node that connects all others
together.

## Update Log

2021-10-07: The latest release “CytoTalk\_v4.0.0” is a completely
re-written R version of the program. Approximately half of the run time
as been shaved off, the program is now cross-compatible with Windows and
\*NIX systems, the file space usage is down to roughly a tenth of what
it was, and graphical outputs have been made easier to import or now
produce portable SVG files with embedded hyperlinks.

2021-06-08: The release “CytoTalk\_v3.1.0” is a major updated R version
on the basis of v3.0.3. We have added a function to generate Cytoscape
files for visualization of each ligand-receptor-associated pathway
extracted from the predicted signaling network between the two given
cell types. For each predicted ligand-receptor pair, its associated
pathway is defined as the user-specified order of the neighborhood of
the ligand and receptor in the two cell types.

2021-05-31: The release “CytoTalk\_v3.0.3” is a revised R version on the
basis of v3.0.2. A bug has been fixed in this version to avoid errors
occurred in some special cases. We also provided a new example
“RunCytoTalk\_Example\_StepByStep.R” to run the CytoTalk algorithm in a
step-by-step fashion. Please download “CytoTalk\_package\_v3.0.3.zip”
from the Releases page
(<https://github.com/huBioinfo/CytoTalk/releases/tag/v3.0.3>) and refer
to the user manual inside the package.

2021-05-19: The release “CytoTalk\_v3.0.2” is a revised R version on the
basis of v3.0.1. A bug has been fixed in this version to avoid running
errors in some extreme cases. Final prediction results will be the same
as v3.0.1. Please download the package from the Releases page
(<https://github.com/huBioinfo/CytoTalk/releases/tag/v3.0.2>) and refer
to the user manual inside the package.

2021-05-12: The release “CytoTalk\_v3.0.1” is an R version, which is
more easily and friendly to use!! Please download the package from the
Releases page
(<https://github.com/huBioinfo/CytoTalk/releases/tag/v3.0.1>) and refer
to the user manual inside the package.

## Citing CytoTalk

-   Hu Y, Peng T, Gao L, Tan K. CytoTalk: *De novo* construction of
    signal transduction networks using single-cell transcriptomic data.
    ***Science Advances***, 2021, 7(16): eabf1356.

    <https://advances.sciencemag.org/content/7/16/eabf1356>

-   Hu Y, Peng T, Gao L, Tan K. CytoTalk: *De novo* construction of
    signal transduction networks using single-cell RNA-Seq data.
    *bioRxiv*, 2020.

    <https://www.biorxiv.org/content/10.1101/2020.03.29.014464v1>

## References

-   Shannon P, et al. Cytoscape: a software environment for integrated
    models of biomolecular interaction networks. *Genome Research*,
    2003, 13: 2498-2504.

## Contact

Kai Tan, <tank1@chop.edu>

<br />
