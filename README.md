# Benchmarking methods for estimating immune cell abundance from bulk RNA-sequencing data

Sturm, G. and Aneichyk T. *Manuscript in preparation.*

The source code in this project can be used to reproduce our results and to use our pipeline for testing additional methods.

## Getting started

### Prerequisites
This pipeline uses [Anaconda](https://conda.io/miniconda.html) and
[Snakemake](https://snakemake.readthedocs.io/en/stable/).

1. **Download and install [Miniconda](https://conda.io/miniconda.html)**
2. **Install snakemake**
```
conda install snakemake
```

3. **Clone this repo.** We use a [git submodule](https://git-scm.com/docs/git-submodule) to import
the source code for the [immundeconv](https://github.com/grst/immunedeconv) R package.
```
git clone --recurse-submodules git@github.com:grst/immune_deconvolution_benchmark.git
```

If you have problems retrieving the submodule, read this [question on
stackoverflow](https://stackoverflow.com/questions/3796927/how-to-git-clone-including-submodules).


### CIBERSORT
Due to licensing restrictions, CIBERSORT could not be included in this repo.
You have to got to the [CIBERSORT website](https://cibersort.stanford.edu),
obtain a license and download the source code.

Place the files `CIBERSORT.R` and `LM22.txt` in the
```
libs/CIBERSORT/
```
folder of this repository.


### Run the pipeline
To perform all computations and to generate a HTML report with [bookdown](https://bookdown.org/yihui/bookdown/) invoke
the corresponding `Snakemake` target:

```
snakemake --use-conda book
```

Make sure to use the `--use-conda` flag to tell Snakemake to download all dependencies from Anaconda.org.




## Test your own method

Our pipeline is designed in a way that you can easily test your own method and benchmark it against the
state-of-the-art. All you have to do is to write an `R` function within the `immunedeconv` package that calls your
method.

Here we demonstrate how to implement and test a method step-by-step using a nonsense random predictor.

In brief, this is what we need to do:

1. Add the new method to the `immunedeconv` package
2. Map the output cell types of the method to the controlled vocabulary
3. Tell the pipeline about the new method
4. Run the pipeline

### Add the new method to the `immunedeconv` package
The sourcecode of the `immunedeconv` package is located in `./immunedeconv`. The pipeline always loads this package from the source code there.

1. **Go to the package and checkout a new branch**

```bash
cd immunedeconv
git checkout -b new_method
```

2. **Edit the file `R/immune_deconvolution_methods.R`**

First, we add our method to the 'list of supported methods':
```r
deconvolution_methods = c("mcp_counter", "epic", "quantiseq", "xcell",
                          "cibersort", "cibersort_abs", "timer",
                          "random") # <- method added here.
```

Next, we add a new deconvolution function for our method.

* Input: gene expression matrix (cols = samples, rows = genes, rownames = HGNC symbols)
* Output: A matrix with immune cell estimates (cols = samples, rows = cell types,
rownames = cell type name)

Note that you can use `system()` to call an arbitrary command line tool.

In our case, we add
```r
#' Deconvolute using the awseome RANDOM technique
#'
#' Here is a good place to add some documentation.
deconvolute_random = function(gene_expression_matrix) {
  # list of the cell types we want to 'predict'
  cell_types = c("CD4+ Tcell", "CD8+ Tcell", "NK cell", "Macrophage",
                 "Monocyte")
  n_samples = ncol(gene_expression_matrix)

  # generate random values
  results = matrix(runif(length(cell_types) * n_samples), ncol=n_samples)

  # rescale the values to sum to 1 for each sample
  results = apply(results, 2, function(x) {x/sum(x)})
  rownames(results) = cell_types

  results
}
```

Finally, register the new method in the generic `deconvolute()` function.
```r
deconvolute.default = function(gene_expression, method=deconvolution_methods, indications=NULL) {
  message(paste0("\n", ">>> Running ", method))
  # run selected method
  res = switch(method,
         xcell = deconvolute_xcell(gene_expression),
         mcp_counter = deconvolute_mcp_counter(gene_expression),
         epic = deconvolute_epic(gene_expression),
         quantiseq = deconvolute_quantiseq(gene_expression),
         cibersort = deconvolute_cibersort(gene_expression, absolute = FALSE),
         cibersort_abs = deconvolute_cibersort(gene_expression, absolute = TRUE),
         timer = deconvolute_timer(gene_expression, indications=indications),
         random = deconvolute_random(gene_expression)   # <- method added here
         )

  # convert to tibble and annotate unified cell_type names
  res = res %>%
    as_tibble(rownames="method_cell_type") %>%
    annotate_cell_type(method=method)

  return(res)
}
```

To check everything, run the unit tests of the `immunedeconv` package.
Invoke the following command from the root of the main repository:
```bash
snakemake --use-conda test_immunedeconv
```

Note that the tests can take some while, and warnings regarding the convergence of EPIC are expected.


### Map the output cell types to the controlled vocabulary
Open the file `immunedeconv/inst/extdata/cell_type_mapping.xlsx` in Excel or
OpenOffice.

Map the cell types to the controlled vocabulary.
* The first column corresponds to the method name
* The second column (`method_cell_type`) corresponds to the cell types, as
  named by the method
* The third column (`cell_type`) corresponds to the corresponding cell type
  entity from the controlled vocabulary.

![screenshot mapping](img/screenshot_mapping.png)



### Run the pipeline

```bash
snakemake wipe   # use this command to clear up previous results and to eradicate the cache
snakemake --use-conda book
```