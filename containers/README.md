# pante

This folder contains the Dockerfiles to build containers to run the pipeline.
The software can be build individually or as a monolithic container, depending on your preference.

Prebuilt released/tagged versions are available at:

- https://hub.docker.com/r/darcyabjones/pante
- https://cloud.sylabs.io/library/darcyabjones/default/pante


To pull those images:


```
singularity pull library://darcyabjones/default/pante:pante-v0.0.1

# or

sudo docker pull darcyabjones/pante:pante-v0.0.1
```

Replacing `pante-v0.0.1` with whatever version you actually want.


The final image contains:

- [bedtools](https://bedtools.readthedocs.io)
- [DECIPHER](http://www2.decipher.codes/)
- [genometools](http://genometools.org/)
- [gffpal](https://github.com/darcyabjones/gffpal)
- [hmmer3](http://hmmer.org/)
- [infernal](http://eddylab.org/infernal/)
- [mitefinder](https://github.com/screamer/miteFinder)
- [MMSeqs](https://github.com/soedinglab/MMseqs2)
- [OcculterCut](https://sourceforge.net/projects/occultercut/)
- [RepeatMasker](http://www.repeatmasker.org/RMDownload.html)
- [RepeatModeler](http://www.repeatmasker.org/RepeatModeler/) (and dependencies RECON, RepeatScout etc.)
- [BLAST+](https://blast.ncbi.nlm.nih.gov/Blast.cgi?CMD=Web&PAGE_TYPE=BlastDocs&DOC_TYPE=Download) with the RepeatMasker "RMBlast" patch.
- [tRNAScan-SE](http://lowelab.ucsc.edu/tRNAscan-SE/)
- [vsearch](https://github.com/torognes/vsearch)

Versions of individual software are available as environment variables and labels in the containers.


## Versioning

The container tags will correspond to a release of the pante pipeline.
They are intended to be paired so that the correct commands are used etc.
It is possible that two docker builds with different tags will be identical because the pipeline has changed but the software dependencies haven't.


## Optimisation

Because MMSeqs2 requires a processor capable of using at least SSE4 instructions, this pipeline also has that minimum requirement.
If you're unsure about your processors capability, on linux you can run the command `grep "sse\|avx" /proc/cpuinfo` to get a list of your CPUs capabilities.

We've done/will do our best to dynamically handle different vectorisation instruction sets in the images where possible.
There should be at least a version that works with SSE and one that works with AVX2, and the correct version should be selected depending on your own processor.

If you really want to squeeze every bit of performance out of whatever you have, you might need to compile the software yourself using the `-march=native` gcc compiler options (or equivalent for your compiler).


## Build system

Because there are a few different bits of software, the container is built locally and pushed rather than using auto-build tools.
Dockerfiles and the Makefile that coordinates building is available at <github.com/darcyabjones/pante/tree/master/containers>.

If you're uncomfortable about running unverified containers you might like to build it yourself.


To build the images you will need [Docker](https://docs.docker.com/install/), [Make](https://www.gnu.org/software/make/), and optionally [Singularity](https://sylabs.io/guides/latest/user-guide/) installed.
You will also need root permission for your computer.

The following commands should work for Linux, Mac, and Linux emulators/VMs (e.g. WSL or Cygwin).

```
# To build all docker images separately.
sudo make docker/all

# To build the monolithic docker image
sudo make docker/pante

# To build a specific dockerfile
sudo make docker/repeatmasker

# To build all singularity images.
sudo make singularity/all

# To build a specific image.
sudo make singularity/mmseqs.sif

# To build the monolithic image.
sudo make singularity/pante.sif

# To tidy the docker image registry and singularity cache.
# Note that this may also remove docker images created by other methods.
sudo make tidy

# To remove all docker images and singularity images from your computer.
# Will remove EVERYTHING, aka. the "help i've run out of root disk space" command.
sudo make clean
```

Singularity images are placed in a `singularity` subdirectory with the extension `.sif`.

Note that singularity images are built from docker images, so you may want to clean up the `docker images`.
Note also that singularity leaves some potentially large tar files in tmp folders that wont be removed automatically.
On my computers (ubuntu/fedora) these are put in `/var/tmp/docker-*`.
It's worth deleting them.


## Proprietary software

RNAmmer has a restricted licence for non-academic users.
This means that it can't be distributed by Docker or singularity hub.

The default pipeline won't run RNAmmer for this reason.

To use RNAmmer you can create the containers for them in this directory.
Note that the order of execution here is important because of the way that make
decides if it needs to rebuild anything.


#### Step 1

Download the source tar file `rnammer-1.2.src.tar.Z` into the `containers/sources` subfolder.
Alternatively, you can provide the `RNAMMER_TAR` variable when you call `make`.

Build the docker container for RNAmmer.

```bash
sudo make docker/rnammer
```

#### Step 2

Pull the pre-build monolithic container from dockerhub.
You could also build the containers yourself, but it takes an hour or two (Because of BLAST).

```bash
sudo make dockerpull/pante

# or
sudo make docker/pante

# or
sudo docker pull darcyabjones/pante-v0.0.1
touch docker/pante  # This is important, otherwise the makefile won't work properly.
```

#### Step 3

Update the monolithic container to include the RNAmmer software.
Which we'll call "pante-plus".

```bash
sudo make docker/pante-plus
sudo make singularity/pante-plus.sif

# OR
# Override the variable
sudo make RNAMMER_TAR="somewhere/else/rnammer-1.2.src.tar.Z" singularity/pante-plus.sif
```

This will create a new monolithic container with RNAmmer for you.
You should be able to run the pipeline with this image instead, and specify `--rnammer` to run the rnammer analysis.

I'll add some documentation about configs later (create an Issue if you need this).


## Using individual containers

There are cases where it might make sense to run each process in a small container
rather than the monolithic container.

There are config profiles called `docker_indiv` and `singularity_indiv` for this purpose.

I don't store the individual containers on dockerhub or singularity hub, because it's a bit onerous to give them nice names and documentation.
You can however just build them from this directory.

To build the docker containers.

```bash
sudo make docker/all

# If you have the rnammer tarball
sudo make docker/rnammer
```

To build the singularity containers

```bash
sudo make singularity/all
sudo make singularity/rnammer.sif
```


## Working with HPC

Lots of HPC systems are beginning to support containerisation.
I'd suggest trying to use the singularity containers most of the time.

Nextflow does support [shifter](https://docs.nersc.gov/programming/shifter/overview/) which would enable you to use docker.
But there are two versions of shifter and Nextflow only supports one version (speak to your sys admins).

If you're still keen on using docker on a system like this where you don't have root-user privileges,
you can convert the images to tarballs and load them back on the other side using `docker save` and `docker load`.

We have a convenience target for this in the Makefile.

So you can run:

```bash
sudo make docker/pante.tar.gz

# OR
sudo make docker/pante-plus.tar.gz
```

Which will put the tarball in the `docker` folder.
