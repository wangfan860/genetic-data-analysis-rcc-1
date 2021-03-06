# Analysis of Genetic Data 1:<br>Making predictions using PCA

Here, we further explore fhe potential of PCA for genetic data by
computing the "projection" of a new set of genotype samples (a "test
set") onto the PCs computed previously from a different set samples
(the "training set").

### A. Ensuring a consistent encoding for the test genotypes

In order to be able to apply our PCA results to a new set of genotype
samples, we need to make sure that they are represented, or *encoded*,
in the genotype matrix *in the exact same way.* (Note: this can be
difficult to achieve, particularly when the genotype data are obtained
using a different genotyping technology, or in different experimental
conditions.) Specifically, we need to make sure that a "0" genotype in
the training samples means the same thing as a "0" genotype in the
test samples, and likewise for genotypes "1" and "2". In this example,
we can enforce the same encoding this using the `--recode-allele`
option in PLINK:

```bash
module load plink/1.90
cd ~/git/gda1/data
cut -f 2,5 1kg_train.bim > alleles.txt
plink --file 1kg_test --recode A-transpose spacex \
--recode-allele alleles.txt --out 1kg_test
less -S 1kg_test.traw
```

:white_check_mark: We now have the genotype data stored as a matrix
with entries that are 0, 1, 2 or missing (NA), and the 0, 1 and 2s for
each SNP mean the exact same thing as the 0, 1 and 2s for each SNP in
the file `1kg_train.traw` which we used to compute the PCs.

### B. Projection as prediction

Next, we open up R again,

```bash
cd ~/git/gda1
R --no-save
```

and we load the genotype data for the test samples into R:

```R
# Load the functions for reading data from various files.
source("code/read.data.R")

# Read the genotypes of the test samples from the .traw file. We also
# use the 1kg_test.fam file here since it is easier to read the sample
# ids from this file.
X <- read.geno.traw(fam.file  = "data/1kg_test.fam",
                    geno.file = "data/1kg_test.traw")
print(X[1:10,1:6])
print(dim(X))
```

The genotype matrix we just loaded should have one row for each of the
29 test samples and one column for each of the 156,923 SNPs (same as
the training samples).

In order to compute the PCA projection for the test samples, we need
to things: (1) the rotation matrix (equivalently, the right-hand
factor of the singular value decomposition); (2) the "mean genotypes"
used to center the matrix. In this next code block we load these two
matrices into R.

```R
# Read the PCA "rotation" matrix generated by MATLAB script geno_pca.m.
R <- read.rot.matrix("results/1kg_train_rot.txt")
print(head(R))
print(dim(R))

# Read the mean genotype vector outputed by geno_pca.m.
y <- read.mean.genotypes("results/1kg_train_mean.txt")
```

The rotation matrix R is a p x k matrix, where k is the number of PCs
computed.

After centering the columns, a simple matrix multiplication completes
the mapping of the test samples onto the embedding defined by the
PCs.

```R
source("code/misc.R")
X           <- center.cols(X,y)
X[is.na(X)] <- 0
pc.test     <- X %*% R
```

The only annoying problem was the small number (<1%) of missing
genotypes. Here we simply set the missing entries to zero after
centering the columns of the matrix, which seems reasonable for lack
of a more sophisticated solution.

Now we will compare the projection of these test samples against the
PCs computed for the training samples. To do so, we will re-generate
plot we created in the [previous episode](02-pca.md), then overlay the
test samples onto this plot. *First, repeat the steps to load and
merge the "pc" data frame containing the PCA results.* Once you have
done this, take the following steps to generate a data frame `pc.test`
for the test samples.

```R
pc.test           <- cbind(data.frame(id = rownames(pc.test)),pc.test)
rownames(pc.test) <- NULL
pc.test           <- merge(panel,pc.test)
pc.test           <- transform(pc.test,id = as.character(id))
print(head(pc.test),digits = 4)
```

We can now use the `plotpca` command to overlay the test samples onto
the plot we made before.

```R
library(ggplot2)
source("code/plotpca.R")
dev.new(height = 6,width = 8)  # Optional.
print(plotpca(pc,1,2,dat.more = pc.test))
```

:ledger: What predictions would you make about the unlabeled genotype
samples based on their projection onto PCs 1 and 2? What predictions
would be most reliable (*i.e.*, most accurate), and what predictions
might be less reliable or less accurate? How does the PCA
visualization inform us about the reliability or accuracy of the
predictions?

:pushpin: In this exercise, consider using the "zoom" feature of
function `plotpca`.

:pushpin: To verify your predictions for the test samples, compare
against the expert-provided population labels by adding the option
`add.labels = TRUE` to your call to `plotpca`.

:pencil2: This demonstrates that we can use PCA applied to genetic
data to make *predictions* about any unseen genotype data as long as
(1) we have genotype data for the same set of SNPs as the training
samples, and (2) the genotypes are encoded in the same way. What is
unrealistic about this "test" of our predictions? In other words, what
additional implicit assumptions are we making about the unseen test
cases?

:pencil2: Apply these same questions to PCs 3 and 4.

:white_check_mark: In [Episode 4](04-admixture.md), we will explore a
different numerical analysis technique that makes very different
assumptions about the genotypes, and as a result it yields different
insights about hidden patterns in the data.
