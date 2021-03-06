---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.12
    jupytext_version: 1.9.1
kernelspec:
  display_name: Python 3
  language: python
  name: python3
---
(sec_tskit_getting_started)=
# Getting started with {program}`tskit`

You've run some simulations or inference methods, and you now have a 
{class}`TreeSequence<tskit.TreeSequence>` object; what now? This tutorial is aimed 
users who are new to {program}`tskit` and would like to get some
basic tasks completed. We'll look at five fundamental things you might
need to do: process trees, sites & mutations, and genotypes, compute statistics, and
export data. Throughout, we'll also provide pointers to where you can learn more.

:::{note}
The examples in this
tutorial are all written in Python, but it's also possible to {ref}`use R <sec_tskit_r>`,
or access the API in [other](https://tskit.dev/tskit/docs/stable/c-api.html)
[languages](https://github.com/tskit-dev/tskit-rust).
:::

First, let's simulate an example tree sequence of a 10Mb chromosome in twenty
diploid individuals using [{program}`msprime`](https://tskit.dev/msprime). To make it
a bit more interesting, we'll simulate the effects of a
{ref}`selective sweep<msprime:sec_ancestry_models_selective_sweeps>` in the middle of
the chromosome, then throw some neutral mutations onto the resulting tree sequence.


```{code-cell} ipython3
import msprime

pop_size=10_000
seq_length=10_000_000

sweep_model = msprime.SweepGenicSelection(
    position=seq_length/2, start_frequency=0.0001, end_frequency=0.9999, s=0.25, dt=1e-6)

ts = msprime.sim_ancestry(
    20,
    model=[sweep_model, msprime.StandardCoalescent()],
    population_size=pop_size,
    sequence_length=seq_length,
    recombination_rate=1e-8,
    random_seed=1234,  # only needed for repeatabilty
    )
# Optionally add finite-site mutations to the ts using the Jukes & Cantor model, creating SNPs
ts = msprime.sim_mutations(ts, rate=1e-8, random_seed=4321)
ts
```

:::{note}
Since we simulated the ancestry of 20 *diploid* individuals, our tree sequence
contains 40 *sample nodes*, one for each genome.
:::

## Processing trees

A common idiom is to iterate over all the {class}`tree objects<tskit.Tree>` in a tree
sequence. This process underlies many tree sequence algorithms, including those we'll
encounter later in this tutorial for calculating 
{ref}`population genetic statistics<tskit:sec_stats>`.
To iterate over your own tree sequence you can use
{meth}`TreeSequence.trees()<tskit.TreeSequence.trees>`. Here, for example, is
how to check whether all trees in the tree sequence have fully coalesced (which is to be
expected in reverse-time, coalescent simulations, but not always for tree sequences
produced by {ref}`forward simulation <sec_tskit_forward_simulations>`).

```{code-cell} ipython3
for tree in ts.trees():
    if tree.has_multiple_roots:
        print("Tree {tree.index} has not coalesced")
        break
else:
    print("All trees in the tree sequence have coalesced")
```

Since all the trees *have* coalesced, at each position in the genome this tree sequence
must have only one most recent common ancestor (MRCA) of the 40 sample nodes. Below, we
iterate over the trees again, finding the IDs of the root (MRCA) node for each tree. The
time of this root node can be found via the {meth}`tskit.TreeSequence.node` method, which
returns a {class}`node object<tskit.Node>` with attributes including the node time:

```{code-cell} ipython3
import matplotlib.pyplot as plt

kb = [0]  # Starting genomic position
mrca_time = []
for tree in ts.trees():
    kb.append(tree.interval.left/1000)  # convert to kb
    mrca = ts.node(tree.root)  # For msprime tree sequences, the root node is the MRCA
    mrca_time.append(mrca.time)
plt.stairs(mrca_time, kb, baseline=None)
plt.xlabel("Genome position (kb)")
plt.ylabel("Time of root (or MRCA) in generations")
plt.yscale("log")
plt.show()
```

It's obvious that there's something unusual about the trees in the middle of this
chromosome, where the selective sweep occurred. It's not particularly efficient to pull
out a tree from the middle of a tree sequence, but it *can* be done, via the
{meth}`TreeSequence.at()<tskit.TreeSequence.at>` method. Here's the tree at the location
of the sweep, drawn using the  {meth}`Tree.draw_svg()<tskit.Tree.draw_svg>` method
(see the {ref}`visualization tutorial <sec_tskit_viz>` for more visualization
possibilities):

```{code-cell} ipython3
from IPython.display import SVG

swept_tree = ts.at(5_000_000)  # or you can get e.g. the first tree using ts.first()
intvl = swept_tree.interval
print(f"Tree number {swept_tree.index}, which runs from position {intvl.left} to {intvl.right}:")
# Draw it at a wide size, to make room for all 40 tips
SVG(swept_tree.draw_svg(size=(1000, 200)))
```

This tree shows the classic signature of a recent expansion or selection event, with many
long terminal branches, resulting in an excess of singleton mutations.

It can often be helpful to slim down a tree sequence so that it represents the genealogy
of a smaller subset of the original samples. This can be done using the powerful
{meth}`TreeSequence.simplify()<tskit.TreeSequence.simplify>` method. Here we will use it
to reduce the number of tips in the trees we are plotting, but it has
{ref}`many other uses<pyslim:sec_left_in_tree_sequence>`.

The {meth}`TreeSequence.draw_svg()<tskit.TreeSequence.draw_svg>` method allows us to draw
more than one tree: either the entire tree sequence, or
(by using the ``x_lim`` parameter) a smaller region of the genome:

```{code-cell} ipython3
reduced_ts = ts.simplify([0, 1, 2, 3, 4, 5, 6, 7, 8, 9])  # simplify to the first 10 samples
print("Genealogy of the first 10 samples, the first 1.5kb of the genome")
display(SVG(reduced_ts.draw_svg(x_lim=(0, 5000))))
```

These are much more standard-looking coalescent trees, with far longer branches higher
up in the tree, and therefore many more mutations at higher-frequencies.

:::{note}
In this tutorial we refer to objects, such as sample nodes, by their numerical IDs. These
can change after simplification, and it is often more meaningful to 
{ref}`work with metadata<sec_tutorial_metadata>`, such as sample and population names,
which can be permanently attached to objects in the tree sequence. Often metadata is
incorporated automatically by the tools generating the tree sequence. Note that if you
wish to edit the tree sequence,
for example to add information such as this, it cannot be done directly, as tree
sequences are immutable. Instead you need to edit a copy of the underlying tables, as
described in the {ref}`data structures tutorial<sec_data_structures>`.
:::



## Processing sites and mutations


For many purposes it may be better to focus on the genealogy of your samples, rather than
the sites and
mutations that
{ref}`define <sec_what_is_dna_data>` the genome sequence itself. This is discussed in the
tutorial entitled "{ref}`sec_tskit_no_mutations`". Nevertheless, {program}`tskit` also
provides efficient ways to return {class}`site objects<tskit.Site>` and
{class}`mutation objects<tskit.Mutation>` from a tree sequence.
For instance, under the finite sites model of mutation that we used above, multiple mutations
can occur at some sites, and we can identify them by iterating over the sites using the
{meth}`TreeSequence.sites()<tskit.TreeSequence.sites>` method:

```{code-cell} ipython3
import numpy as np
num_muts = np.zeros(ts.num_sites, dtype=int)
for site in ts.sites():
    num_muts[site.id] = len(site.mutations)  # site.mutations is a list of mutations at the site

# Print out some info about mutations per site
for nmuts, count in enumerate(np.bincount(num_muts)):
    info = f"{count} sites"
    if nmuts > 1:
        info += f", with IDs {np.where(num_muts==nmuts)[0]},"
    print(info, f"have {nmuts} mutation" + ("s" if nmuts != 1 else ""))
```

## Processing genotypes

At each site, the sample nodes will have a particular allelic state (or be flagged as
{ref}`tskit:sec_data_model_missing_data`). The
{meth}`TreeSequence.variants()<tskit.TreeSequence.variants>` method gives access to the
full variation data. For efficiency, the {attr}`genotypes <tskit.Variant.genotypes>`
at a site are returned as a [numpy](https://numpy.org) array of integers:

```{code-cell} ipython3
import numpy as np
np.set_printoptions(linewidth=200)  # print genotypes on a single line

print("Genotypes")
for v in ts.variants():
    print(f"Site {v.site.id}: {v.genotypes}")
    if v.site.id >= 4:  # only print up to site ID 4
        print("...")
        break
```

:::{note}
Tree sequences are optimised to look at all samples at one site, then all samples at an
adjacent site, and so on along the genome. It is much less efficient look at all the
sites for a single sample, then all the sites for the next sample, etc. In other words,
you should generally iterate over sites, not samples. Nevertheless, all the alleles for
a single sample can be obtained via the
{meth}`TreeSequence.haplotypes()<tskit.TreeSequence.haplotypes>` method.
:::


To find the actual allelic states at a site, you can refer to the
{attr}`alleles <tskit.Variant.alleles>` provided for each {class}`variant<tskit.Variant>`:
the genotype value is an index into this list. Here's one way to print them out; for
clarity this example also prints out the IDs of both the sample nodes (i.e. the genomes)
and the diploid {ref}`individuals <sec_nodes_or_individuals>` in which each sample
node resides.

```{code-cell} ipython3
samp_ids = ts.samples()
print("  ID of diploid individual: ", " ".join([f"{ts.node(s).individual:3}" for s in samp_ids]))
print("       ID of (sample) node: ", " ".join([f"{s:3}" for s in samp_ids]))
for v in ts.variants():
    site = v.site
    alleles = np.array(v.alleles)
    print(f"Site {site.id} (ancestral state '{site.ancestral_state}')",  alleles[v.genotypes])
    if site.id >= 4:  # only print up to site ID 4
        print("...")
        break
```

:::{note}
Since we have used the {class}`msprime.JC69` model of mutations, the alleles are all
either 'A', 'T', 'G', or 'C'. However, more complex mutation models can involve mutations
such as indels, leading to allelic states which need not be one of these 4 letters, nor
even be a single letter.
:::


## Compute statistics

There are a {ref}`large number of statistics<tskit:sec_stats>` and related calculations
built in to {program}`tskit`. Indeed, many basic population genetic statistics are based
on the allele (or site) frequency spectrum (AFS), which can be obtained from a tree sequence
using the {meth}`TreeSequence.allele_frequency_spectrum<tskit.TreeSequence.allele_frequency_spectrum>`
method:

```{code-cell} ipython3
afs = ts.allele_frequency_spectrum()
plt.bar(np.arange(ts.num_samples + 1), afs)
plt.title("Unpolarised allele frequency spectrum")
plt.show()
```

By default this method returns the "folded" or unpolarized AFS that doesn't
{ref}`take account of the ancestral state<tskit:sec_stats_polarisation>`.
However, since the tree sequence provides the ancestral state, we can plot the polarized
version; additionally we can base our calculations on branch lengths rather than alleles,
which provides an estimate that is not influenced by random mutational "noise".

```{code-cell} ipython3
fig, (ax1, ax2) = plt.subplots(ncols=2, figsize=(12, 3))

afs1 = ts.allele_frequency_spectrum(polarised=True, mode="branch")
ax1.bar(np.arange(ts.num_samples+1), afs1)
ax1.set_title("Genome-wide branch-length AFS")

restricted_ts = ts.keep_intervals([[5e6, 5.5e6]])
afs2 = restricted_ts.allele_frequency_spectrum(polarised=True, mode="branch")
ax2.bar(np.arange(restricted_ts.num_samples+1), afs2)
ax2.set_title("Branch-length AFS between 5 and 5.5Mb")

plt.show()
```

On the left is the frequency spectrum averaged over the entire genome, and on the right
is the spectrum for a section of the tree sequence between 5 and 5.5Mb, which we've
created by deleting the regions outside that interval using
{meth}`TreeSequence.keep_intervals()<tskit.TreeSequence.keep_intervals>`. Unsurprisingly,
as we noted when looking at the trees, there's a far higher proportion of singletons in
the region of the sweep.

It is often useful to see how statistics vary in different genomic regions. This is done
by calculating them in {ref}`tskit:sec_stats_windows` along the genome. For this,
let's look at a single statistic, the genetic diversity (π). As a site statistic this
measures the average number of genetic differences between two randomly chosen samples,
whereas as a branch length statistic it measures the average branch length between them.
We'll plot how the value of π changes using 10kb windows, plotting the resulting
diversity between positions 4 and 6 Mb:

```{code-cell} ipython3
fig, (ax1, ax2) = plt.subplots(ncols=2, figsize=(12, 3))
L = int(ts.sequence_length)
windows = np.linspace(0, L, num=L//10_000)
ax1.stairs(ts.diversity(windows=windows), windows/1_000, baseline=None)  # Default is mode="site"
ax1.set_ylabel("Diversity")
ax1.set_xlabel("Genome position (kb)")
ax1.set_title("Site-based calculation")
ax1.set_xlim(4e3, 6e3)
ax1.set_yscale("log")
ax2.stairs(ts.diversity(windows=windows, mode="branch"), windows/1_000, baseline=None)
ax2.set_xlabel("Genome position (kb)")
ax2.set_title("Branch-length-based calculation")
ax2.set_xlim(4e3, 6e3)
ax2.set_yscale("log")
plt.show()
```

There's a clear drop in diversity in the region of the selective sweep. And as expected,
the statistic based on branch-lengths gives a less noisy signal.


## Exporting data

:::{todo}
Saving in VCF or ms format.
:::

## Key points covered above

Some simple methods and take-home messages from this tutorial, in rough order of importance:

* In python, a tree sequence object has a number of basic attributes such as
    ``.num_trees``, ``.num_sites``, ``.num_samples``, ``.sequence_length``. Similarly
    a tree object has e.g. an ``.interval`` attribute, a site object has a ``.mutations``
    attribute, a node object has a ``.time`` attribute, and so on.
* Each sample node corresponds to a sampled genome; diploid individuals in a tree sequence
    contain 2 sample nodes
* ``ts.samples()`` returns an array of node IDs specifying the nodes that are marked as samples
* ``ts.node(X)`` returns the node object for node ID X
* ``ts.trees()`` iterates over all the trees
* ``ts.sites()`` iterates over all the sites
* ``ts.variants()`` iterates over all the sites with their genotypes and alleles
* ``ts.simplify()`` reduces the number of sample nodes in the tree sequence to a specified subset
* ``ts.keep_intervals()`` (or its complement, ``ts.delete_intervals()``) deletes unwanted parts of the genome
* ``.draw_svg()`` plots trees or tree sequences
* ``ts.at()`` returns a tree at a particular genomic position (but using ``ts.trees()`` is usually preferable)
* Various population genetic statistics can be calculated using methods on a tree sequence, for example
    ``ts.allele_frequency_spectrum``, ``ts.diversity``, ``ts.Fst``, and these can also be
    calculated in windows along the genome.

