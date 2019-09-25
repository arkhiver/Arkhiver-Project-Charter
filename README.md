# Design proposal for the Arkhiver project

## Overview

**Arkhiver** is a tool to create reproducible computational environments and execution pipelines for a variety of popular programming languages.

Although it will be a generic open-source project, the first beneficiary is probably going to be the Journal of Open Source Economics (JOSEcon), where it is going to help in automatizing the paper submission and review process. Furthermore, it aims to provide a low-cost way for researchers to start building on each other's code, hopefully resulting in more structured, maintainable and bug-free algorithms in the scientific community.

Arkhiver is built upon the [repo2docker](https://github.com/jupyter/repo2docker) project,
which in itself is capable of building, running and publishing reproducible computational environments.

However, for the submission and review process, three relevant features are missing:
 - defining reproducible run configurations or "pipelines"
 - generation of easy-to-consume run reports allowing comparison of reproduced results with a reference resultset
 - long-term archival of the run environment & pipeline configuration

### Pipelines
A pipeline is simply a sequence of automated steps to be carried out on the built docker image. While Arkhiver provides a generic and flexible interface for pipeline definition, the following examples of pipelines would be typical for scientific reproducibility:

- **Run a sequence of scripts/commands to verify the environment (minimal proof of run)**
These scripts demonstrate that the environment is correctly set up, and the code is ready for experimentation.
In the submission process, the criteria would be that this pipeline succeeds, before any human reviewers are assigned
to the paper.

- **Run a sequence of scripts/commands to produce outputs (proof of correctness)**
In case the paper is a code supplement for a particular conventional paper, this pipeline could, e.g. generate all the results
the authors have used in the accompanying conventional article. Consequently, the review process is tractable and simpler to carry out.

### Testing outputs

Pipeline outputs can be useful for another reason: to test different builds of the code and compare outputs with a reference resultset.

Most popular programming languages rely heavily on dynamic dependency resolution and lots of third-party packages and add-ons. A build today and six months from now on the same repository might not yield the same binaries. Furthermore, if the author fails to properly encapsulate the sequence of steps required to achieve a particular output, then the published results are not reproducible.

To help early discovery of such discrepancies, Arkhiver adds the feature to **compare outputs** generated by a local build with some reference expected outputs, placed inside the source repository.

Comparing results can be complex, so only specific file types are going to be recommended:

- CSV/Plain text files
- Image files (near-duplicate detection)
- Jupyter notebooks

The goal of arkhiver is to produce easy-to-use **interactive HTML reports** which provide a simple self-contained document contrasting each generated artefact with its reference value in an easy to consume fashion.

### Long-term archival

GitHub provides an easy way to share sourcecode while Docker registries offer an easy way for sharing Docker images.

However, when it comes to scientific results, we're usually looking for long-term archival.

Arkhiver provides a way to package your executable images into standalone files that you can deposit at service providers like [Zenodo](https://zenodo.org/).

## The arkhiver package

Arkhiver is based on the original `repo2docker` repository, with the top level executable `arkhiver`.

On top of the features provided by `repo2docker`, Arkhiver would provide the following features:

1. Reproducible Execution Pipeline Specification (REPS)

Similary to repo2docker's [Reproducible Execution Environment Specification](https://repo2docker.readthedocs.io/en/latest/specification.html#the-reproducible-execution-environment-specification),
Arkhiver supports defining various pipelines in a special `pipelines.yaml` file. The Arkhiver documentation contains examples and complete documentation for the pipeline definition.

An example for a valid `pipelines.yaml` file would be:

``` yaml
minimal:
  script:
    - python demo_preprocess.py
    - jupyter nbconvert --execute demo.ipynb

complete:
  script:
    - python preprocess.py
    - jupyter nbconvert --execute complete.ipynb
  time: 2 days
  compare:
    - complete.ipynb
    - summary.csv
```

In the above example, there are 2 pipelines: `minimal` and `complete`. Each pipeline may have 3 keywords defined: `script`, `time` and `compare`. The following table summarizes the effects the different fields have on the different run commands:


| Command | Keyword | Required | Effect |
|----|----|----|----|
| run | script | yes | The sequence of commands to execute |
| run | time | no | The time limit after the run is marked as failure |
| run | compare | no | Sequence of files marked for comparison. In the run step, these are copied to a safe location, and the code is supposed to overwrite the original files |
| compare | compare | no | Sequence of files marked for comparison. In the compare step, each file produced by the code is compared with the originals with the appropriate diff tool | 
 
In the above example, one can check if the built image runs as expected with

``` bash
arkhiver run minimal <built image>
```
To reproduce the complete set of results, execute

``` bash
arkhiver run complete <built image> <output dir>
```

Finally, to compare your local results with the reference ones & generate a Markdown/HTML report

``` bash
arkhiver compare complete <built image> <output dir>
```

In the compare section, the contents of before and after states of the specified files are computed (the reference outputs are part of the docker image, which get contrasted by the new outputs). The differences are summarized in a generated interactive html report, where each file comparison in the compare list is displayed in a different tab. For notebooks and text files, standard tools like `nbdiff` (nbdime) and `diff` will be used to produce a visual diff output. For binary content (e.g. images), if they are not byte identical, the before and after states are displayed side-by-side & image comparison tools (e.g. slider) can be provided.

2. Build logs

In theory, the [Reproducible Execution Environment Specification](https://repo2docker.readthedocs.io/en/latest/specification.html#the-reproducible-execution-environment-specification), should be enough to set up the environment on a native machine. Sometimes this is difficult due to e.g. OS library dependencies. In these cases, a log of the docker build can be quite handy to track what dependencies were installed. The build logs of `repo2docker` are copied into the final image by `arkhiver`, to `/var/log/repo2docker`.

3. Distributing images as files

For scientific results it is standard procedure to store both the source (the code) and the results (the pipeline outputs) in cloud storage. Arkhiver makes this easy through the `package` and `load` commands, which pack and unpack built images into standalone files. These commands are basically thin wrappers around `docker save` and `docker load`.

``` bash
arkhiver package/load <built image> <output>
```

---
2019 Sep 6, Alphacruncher AG.
