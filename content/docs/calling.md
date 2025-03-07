+++
title = "Calling variants"
weight = 2
+++

Varlociraptor allows to call small and structural variant events in different scenarios.
Note that varlociraptor requires to provide a set of candidate variants to consider.
By this, our model becomes independent of denovo variant detection mechanisms and can be combined with any caller (e.g. GATK, Freebayes, Delly, ...). In particular, you can also first combine candidates from different callers, to cover diverse variant types and length ranges.
Let `candidates.bcf` be the set of candidate variant calls (VCF or BCF format, see [bcftools](https://samtools.github.io/bcftools/)).
Let `reference.fa` be the reference genome FASTA file (indexed with [samtools](https://www.htslib.org/doc/samtools.html)).

## Preprocessing per-sample observations

First, varlociraptor requires to preprocess the candidate variants in order to obtain per-sample observations for the actual calling process.
Let `sample.bam` be the aligned reads of the sample to preprocess.
While varlociraptor does not require a particular read aligner, it assumes that provided mapping qualities (MAPQ) are as accurate as possible, and that too large indels are encoded as softclips (i.e. it expects BAMs in [bwa mem](https://bio-bwa.sourceforge.net/) style).
Preprocessing can be started with

```bash
varlociraptor preprocess variants reference.fa --bam sample.bam < candidates.bcf > sample.observations.bcf
```

In other words, the candidate variants are piped into varlociraptor (with the `<` shell operator) and observations are piped into `sample.observations.bcf` (with the `>` shell operator).

Note that the candidate variants (here ``candidates.bcf``) have to be **the same for each sample**.
This can be achieved by either jointly calling across all samples (as possible with most variant callers), or by merging candidates from all involved samples into a single VCF/BCF file.
From the candidate variant file, only `CHROM`, `POS`, `REF`, `ALT`, `EVENT` (in case of breakends), `END`, and `SVLEN` (if present) are used. Other fields are ignored.
Variant types that are not (yet) supported by Varlociraptor will be dropped (with notification on STDERR).

### Parallelization

Instead of offering internal parallelization, ``varlociraptor preprocess variants`` should be parallelized via a scatter-gather.
That means that you first split the ``candidates.bcf`` into chunks of equal size.
This can best be done via [Rust-Bio-Tools](https://github.com/rust-bio/rust-bio-tools) (``rbt``), which offers a subcommand for splitting VCF/BCF files while properly handling events that span multiple records (breakend events with ``SVTYPE=BND``).
For example, with

```bash
rbt vcf-split candidates.bcf chunk{0..15}.bcf
```

the ``candidates.bcf`` is splitted into 16 chunks of approximately equal size.
After applying ``varlociraptor preprocess variants`` and also the calling steps described below (and, depending on the research question, a variant annotation tool like VEP), the processed chunks can be merged again.

If the input candidates did contain breakend variants (``SVTYPE=BND``, which only happens when generated by structural variant callers), they have to be sorted first e.g., via ``bcftools sort``.
Merging can happen via ``bcftools concat``.

### Fast mode

Preprocessing is the most time-intensive part for variant calling with varlociraptor.
However, it only has to be done once for each sample, and can be re-used for different variant calling steps.
In case the exact allele frequency is less important for you (e.g. with large cohorts), you can add the argument ``--pairhmm-mode fast``.
This will approximate the pair HMM computation of Varlociraptor by just considering the optimal alignment path, thereby changing the computation of the allele likelihoods from quadratic to linear.
In rare cases this can lead to wrong probabilities for single reads though, such that allele frequencies can become slightly incorrect.

## Calling

The basic calling command is

```bash
varlociraptor call variants ...
```

The dots (`...`) refer to the selected calling mode, which can be one of the following:

1. **Tumor-normal variant calling:** this assumes that a tumor and a corresponding healthy sample is given.
2. **Generic variant calling:** via a *variant calling grammar*, arbitrary calling scenarios can be defined.

### Note on amplicon sequencing data

With amplicon sequencing data, variants are usually covered only by a few (down to a single) amplicon, from which many duplicates are sequenced.
It important to deactivate read position bias handling via

```bash
varlociraptor call variants --omit-read-position-bias ...
```

in the Varlociraptor model in such cases, because it will otherwise detect a read position bias for those variants covered only by a single amplicon. Even for few amplicons the model would otherwise overestimate the degrees of freedom, thereby leading to artificially weak probabilities.

## Tumor-normal variant calling

Let `tumor.bcf` and `normal.bcf` be the preprocessed observations (see above) of the tumor and healthy/normal sample, respectively.
Then, variants of all lengths can be called with

```bash
varlociraptor call variants tumor-normal --purity 0.75 --tumor tumor.bcf --normal normal.bcf > calls.bcf
```

with `0.75` being the purity (i.e. the cancer cell content) of the tumor sample.
The result is a proper stastistical assessment of the somatic and germline variants, without the need to apply any ad-hoc filtering.
Instead, it should be followed by controlling the false discovery rate over the desired events, see [Filtering]({{< ref "filtering.md" >}}).

## Generic variant calling

The generic mode allows to define a calling scenario via a variant calling grammar.
The grammar allows to define the desired scenario in a [YAML](https://yaml.org) file.
For example, the following defines a normal/tumor/relapse model with a uniform prior (i.e., joint calling of tumor and relapse after therapy against normal):

```yaml
samples:
  normal:
    resolution: 5
    universe: "0.0 | 0.5 | 1.0 | ]0.0,0.5["
  tumor:
    resolution: 100
    universe: "[0.0,1.0]"
    contamination:
      by: normal
      fraction: 0.25
  relapse:
    resolution: 100
    universe: "[0.0,1.0]"
    contamination:
      by: normal
      fraction: 0.53

events:
  germline:        "normal:0.5 | normal:1.0"
  somatic_normal:  "normal:]0.0,0.5["
  somatic_tumor:   "normal:0.0 & tumor:]0.0,1.0]"
  somatic_relapse: "normal:0.0 & tumor:0.0 & relapse:]0.0,1.0]"
```

In the following, we briefly describe each element for the grammar.

* `samples`: this section contains the definition of the involved samples. Each sample is listed by its name (e.g. `normal`) which is referred to later in the `events` section.
* `resolution`: the number of points in allele frequency space to evaluate when integrating over continuous allele frequency intervals (e.g. `]0.0,0.5[`). A resolution of 100 in an allele frequency interval of `]0.0,1.0[` means that allele frequency is evaluated in steps of size `0.01`.
* `universe`: valid allele frequencies in the given sample. The operator `|` denotes a logical "or". For example `0.0 | 0.5 | 1.0 | ]0.0,0.5[` means that an allele frequency of `0.0`, `0.5`, `1.0` or any frequency in the interval `]0.0,0.5[` (with exlcusive bounds) is possible for the particular sample. Defining a `universe` means that a uniform prior is used. Alternatively, when the mutation rates are known, it is possible to configure Varlociraptors joint prior distribution that allows to model population genetics, mendelian inheritance and tumor evolution. See the next section for details.
* `contamination`: denotes the contamination of the sample with another sample, given by its name after the `by` key, and the fraction of contamination after the `fraction` key.
* `events`: this section contains the definition of events that shall be evaluated. Each event is a boolean logic formula over operands that define allele frequencies or allele frequency intervals in particular samples. These operands have the form `samplename:spec`, where spec is the specification of an allele frequency (e.g. `0.5`) or an allele frequency interval (inclusive: `[a,b]`, left-exclusive: `]a,b]`, right-exclusive: `[a,b[`, exclusive: `]a,b[`).
* `expressions`: analogous to `events`, expressions allow to define formulas, each with a given name. However, expressions are by default ignored, and have to be explicitly used from within `events` formulae. For example, let `myexpr` be the name of an expression, then it can be used in any formula by specifying `$myexpr`.

For each variant, Valrociraptor will calculate the probability of each defined event to be true.
Importantly, for proper results, the given events have to **cover the entire range of possibilities**. 
Otherwise, the calculated posterior probabilities would be biased.
The absent event (i.e. here `tumor:0.0 & normal:0.0 & relapse:0.0`) is added implicitly though.

Let `scenario.yaml` be the defined calling scenario (e.g., as above).
Let `relapse.bcf`, `tumor.bcf`, and `normal.bcf` be the preprocessed observations of relapse, tumor and healthy/normal sample, respectively, as defined above.
Then, the calling command would be

```bash
varlociraptor call variants generic --scenario scenario.yaml --obs relapse=relapse.sorted.bcf tumor=tumor.bcf normal=normal.bam > calls.bcf
```

Note that now, observation files are given with a leading name, which has to correspond to the name defined in the scenario YAML.
The result is a proper stastistical assessment of the desired scenario, without the need to apply any ad-hoc filtering.
Instead, it should be followed by controlling the false discovery rate over the desired events, see [Filtering]({{< ref "filtering.md" >}}).

### Configuring the joint prior distribution

When mutation rates for the species you investigate (or the tumor) are known, it is possible to inform Varlociraptor about such prior knowledge, including inheritance relations. 
Which sufficient evidence, such prior knowledge is less important, however, it can play a role in corner cases (in particular at low coverage), and it can help the system to make the correct decision in case of ambiguity.
In the following example, instead of defining uniform allele frequency universes as above, we define the inheritance relationship between samples and the properties of the underlying species.

```yaml
species:
  genome-size: 3.5e9
  heterozygosity: 0.001
  germline-mutation-rate: 1e-3
  ploidy:
    male:
      all: 2
      X: 1
      Y: 1
    female:
      all: 2
      X: 2
      Y: 0

samples:
  mother:
    sex: female
  father:
    sex: male
  daughter:
    sex: female
    somatic-effective-mutation-rate: 1e-10
    inheritance:
      mendelian:
        from:
          - mother
          - father
  tumor:
    sex: female
    somatic-effective-mutation-rate: 1e-6
    inheritance:
      clonal:
        from: daughter
        somatic: false
    contamination:
      by: normal
      fraction: 0.1

events:
  germline_denovo:    "(daughter:0.5 | daughter:1.0) & father:0.0 & mother:0.0"
  somatic_normal:     "daughter:]0.0,0.5["
  somatic_tumor:      "daughter:0.0 & tumor:]0.0,1.0]"
  germline_inherited: "!daugher:0.0 & (!father:0.0 | !mother:0.0)"
  germline_lost:      "daughter:0.0 & (!father:0.0 | !mother:0.0)
```

Compared to the initial normal/tumor/relapse with a uniform prior example shown above, some new elements appear:

* `species`: here, properties of the underlying species are defined, that is, the genome size (needed for the somatic prior), the heterozygosity (fraction of expected heterozygous sites), the germline mutation rate (for de novo mutations during the mendelian inheritance process), the ploidy for each sex. For the latter, `all` denotes the ploidy of all chromosomes, while below special cases are listed, e.g., for `X` and `Y` chromosomes.
* `sex`: denotes the sex of each sample, thereby defining the ploidy, which is taken from the species definition.
* `somatic-effective-mutation-rate`: denotes the expected rate of de novo somatic mutations that survive (i.e. aren't lethal) to formulate a prior assumption about the expected somatic allele frequency distribution (as presented by [Williams et al. Nature Genetics 2016](https://doi.org/10.1038/ng.3489)). Such rates will usually differ between normal (can be found in literature) and tumor samples, and should be estimated or looked up for a given tumor instance. Note that it is also possible to simply define an allele frequency `universe` for a tumor sample, thereby avoiding to specify a somatic effective mutation rate, in case it is still unknown. In that case, a uniform prior would be used for the tumor, while the rest of the sample would be treated according to the defined population genetic and mendelian priors.
* `inheritance`: Relationship to other samples. Inheritance can be either `mendelian` (here, the `daughter` inherits from `father` and `mother`), or `clonal` (here, the `tumor` inherits from the normal tissue of the `daughter`). In case of the latter, it has to be specified additionally whether somatic mutations from the parental clone are inherited or not.

All other keywords can be looked up in the section above.

### Grammar applications

Various concrete applications of the grammar can be found in the [Varlociraptor Scenario Catalog](https://varlociraptor.github.io/varlociraptor-scenarios).
You are welcome to submit further applications there.

## Supported variant types

Varlociraptor implements support for all kinds of variants in all length ranges

* SNVs,
* MNVs (multiple nucleotide variants, as called by e.g. Freebayes),
* replacements (small and large),
* Insertions (small and large),
* Deletions (small and large),
* Inversions (small and large),
* Duplications (small and large),
* Breakends.
