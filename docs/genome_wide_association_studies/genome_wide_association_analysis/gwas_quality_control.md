---
sidebar_position: 2
---

# GWAS quality control

We will now work with a set of data files containing many SNPs from chromosome 19 genotyped on controls and cases. Data from a
GWAS would contain SNPs at this density across the entire genome, but we will focus on just one chromosome to make the
exercises more tractable.

The key files are `chr19-example.vcf.gz` and `chr19-example.samples`. If you followed the [getting started
section](./gwas_analysis_plink.md), you should already have these in your folder.

Before getting started, let's convert these files from VCF format to a 'binary PED' format that PLINK can read more easily:

```sh
./plink --make-bed \
--vcf chr19-example.vcf.gz \
--pheno chr19-example.samples \
--update-sex chr19-example.samples 2 \
--out chr19-example \
--keep-allele-order
```

This might take a moment to run.  You should see plink has created three new files:

    chr19-example.bed
    chr19-example.bim
    chr19-example.fam

These files contain information about the samples and SNPs, as well as the genotypes for each of the samples at each of the
SNPs. You can look at the `.bed` and `.fam` files using `less`, but the `.bed` file is in a special binary format that only
plink and other software tools can understand.

:::tip Note

We added the `--keep-allele-order` option here to the above command to make sure plink preserves the two alleles at each SNP in
the same order as in the VCF file. This is **important** because we will want to be able to keep track of the direction of
estimated effects.

:::

We can tell `plink` to load data in the binary plink format using the `--bfile` option. For instance, to calculate allele
frequencies we now use:

```sh
./plink --bfile chr19-example --freq --out chr19-example
```

::tip Question
How many SNPs and samples are in this dataset?
:::

:::note Note

Many other formats are in use for genetic data, including 'GEN' format, BCF (binary VCF) format, and ['BGEN'
format](http://www.bgenformat.org). Learning how to deal with these is part of the job description.

:::

There are many different QC metrics that we can calculate for our dataset. These metrics can tell us about the quality of loci
(i.e. SNPs), and of samples. For instance, we can calculation information about missingness:

```sh
./plink --bfile chr19-example --missing --out miss-info
```

This will produce a `.imiss` file with information about individuals and .lmiss with information about loci (SNPs). You can load
this output into a spreadsheet program to look at it in more detail: In the directory browser right-click miss-info.lmiss
file. Select open in Libreoffice Calc or choose other application to select Libreoffice Calc. An options window will open and
in the Separator options you need to ensure that only 'separated by space' and the 'merge delimiters' boxes are checked before
proceeding!

:::tip Question
Which SNP has the highest missing rate?
:::

Another QC metric is sample heterozygosity:

```sh
./plink --bfile chr19-example --het --out het-info
```

We can use R to visualize these results. Launch an RStudio session now and make sure it is working in the `gwas_practical`
directory.  If you followed the instructions above this should work:
```
setwd( '/Users/<your user name>/Desktop/' )
```
(or you can use the `Session -> Set Working Directory` memu option.)

Now let's load the QC results into R and plot them:
```R
# In R:
HetData <- read.table( "het-info.het", h=T )
hist(
    HetData$F,
    breaks=100,
    xlab = 'F values',
    ylab = "Count"
)
```


The graph you see shows the distribution of heterozygosity (or more accurately the **inbreeding coefficient**) across samples.
QC plots often look like this; there is a large peak of samples that lie within a range of normal values (the normal samples),
and then a small number of outlier samples that are usually poor quality samples. To improve the plot without losing any data,
let's *clamp* the values into a smaller x axis range:
```
clamp <- function( x, lower = -Inf, upper = Inf ) {
    pmax( pmin( x, upper ), lower )
}
hist(
    clamp( HetData$F, -0.2, 0.5 ),
    breaks=100,
    xlab = 'F values',
    ylab = "Count"
)
```

You can see that there is a large peak in heterozygosity around 0.1, with a number of outliers below 0 or above 0.2.

:::tip Saving images

In Rstudio you can save any figure for later reference - go to the 'Export' button above the plot and choose a destination
file.  Alternatively you can save it directly using the `png()` command like this:
```R
# In R:
png( "HetHist.png" )
hist(
    clamp( HetData$F, -0.2, 0.5 ),
    breaks=100,
    xlab = 'F values',
    ylab = "Count"
)
dev.off()
```
:::

Next, lets plot the missingness values.

```R
# In R:
MissData <- read.table( "miss-info.imiss", header = T )
hist(
    MissData$F_MISS,
    breaks=100,
    xlab = "Missing proportion",
    ylab = "Count"
)
```

That's again not very informative because of the scale... let's zoom in to the region near zero on the x axis by clamping
values to 0.2:

```
hist(
    clamp( MissData$F_MISS, upper = 0.2 ),
    breaks=100,
    xlab = "Missing proportion",
    ylab = "Count"
)
```

Again, most of the samples have low missingness (close to zero), with a number of outliers above 0.02. You can see some other
features there as well such as the two bumps in the distribution which in a real analysis we'd want to investigate as well.
However for now, let's combine the two QC metrics (missingness and heterozygosity) to select outlying samples:

```R
# In R:
qcFails <- MissData[
    MissData$F_MISS > 0.02
    | HetData$F > 0.2
    | HetData$F < 0,
    c(1:2)                   # Just capture the sample identifier fields
]
write.table(
    qcFails,
    file = "qcFails.txt",
    quote = FALSE,
    row.names = FALSE,
    col.names = FALSE,
)
```

Now let's combine this list with other quality control metrics to create a clean dataset.

We will:

* filter samples ou that have outlying missingness or heterozygosity, as above,

* and filter out SNPs based on SNP QC metrics using the `--geno`, `--hwe`, and `--maf` options.

Here is a plink command that does this:

```sh
./plink \
--bfile chr19-example \
--remove qcFails.txt \
--hwe 1e-4 include-nonctrl \
--geno 0.01 \
--maf 0.01 \
--make-bed \
--out chr19-clean \
--keep-allele-order
```

:::warning Note

The backslash characters above are 'line continuation characters' - they're just there to make the command work as if we typed
it all one one line.

:::

:::tip Questions

What SNP QC filters have we applied here? (To find out, see the
[documentation on --hwe](https://www.cog-genomics.org/plink/1.9/filter#hwe),
on [--geno](https://www.cog-genomics.org/plink/1.9/filter#missing),
and on [--maf](https://www.cog-genomics.org/plink/1.9/filter#maf)).

Read the output of the command carefully. How many samples did the command retain? Check in R this is the expected number. And
how many SNPs did we keep - does this number look sensible?

:::

### Next steps

Congratulations!  You now have a QC'd dataset to work with.  Now go on to the [association practical](./gwas_association/).