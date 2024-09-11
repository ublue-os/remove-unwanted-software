# Maximize available disk space for build tasks

Modified version of [easimon/maximize-build-space], this fork only removes unwanted software from GitHub runner CI.

By default, public shared Github runners come with around 25-29 GB of free disk space to be consumed by your build job.
If this is too little for you, and your build job runs out of disk space, this action might be for you.

If not: please go on, there's nothing to see here.

This action maximizes the available disk space on public shared GitHub runners, by

- ~~Utilizing space on an otherwise unused temp disk~~
- Optionally removing unnecessary preinstalled software
- Optionally removing the swapfile

This shall make you gain around 32GB on Ubuntu 22.04

The idea came up when I tried to build a Ubuntu Linux Kernel DEB on Github, and the Job aborted due to disk space issues.

## Caveats

- **This action is a hack, really**, and on a "works for me" basis. It has been built by modifying [easimon/maximize-build-space] and reverse-enginnering undocumented parts of the Github runner setup, which might change in the future -- up to the point where this action ceases to work. For sure you're voiding the warranty on Github runners when using this. **If you are not running into disk space issues with the default runner setup, don't use it.**.
- Removal of unnecessary software is currently implemented by `rm -rf` on specific folders, not by using a package manager or anything sophisticated. While this is quick and easy, it might delete dependencies that are required by your job and so break your build (e.g. because your build job uses a .NET based tool and you removed the required runtime).

Btw: the choice of removable packages is not an expression of dislike against these -- they were just the largest folders for a single toolkit which @easimon could find quickly.

## How it works

At the time of writing, public [Github-hosted runners](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners) are using [Azure DS2_v2 virtual machines](https://docs.microsoft.com/en-us/azure/virtual-machines/dv2-dsv2-series#dsv2-series), featuring a 84GB OS disk on `/` and a 14GB temp disk mounted on `/mnt`.

1. The root file system has ~29GB (of 84GB) available, the rest being consumed by the preinstalled build environment. Github runners come with a rich choice of software, see the image descriptions for [Ubuntu 18.04](https://github.com/actions/virtual-environments/blob/main/images/linux/Ubuntu1804-README.md) or [Ubuntu 20.04](https://github.com/actions/virtual-environments/blob/main/images/linux/Ubuntu2004-README.md). This is great to support a wide variety of workflows out of the box, but also consumes a lot of space by providing programs that might be unnecessary for an individual build job.

This action does the following:

1. (Optionally) removes unwanted preinstalled software and the swapfile
1. ~~Concatenates the free space on `/` and `/mnt` (the temp disk) to an LVM volume group~~
1. ~~Creates a swap partition and a build volume on that volume group~~
1. ~~Mounts the build volume back to a given path (`${GITHUB_WORKSPACE}` by default)~~

This results in the space of the previously installed packages (around 19GB) available for your build job.

## Usage

You should most probably use this action as the first build step, even **before** `actions/checkout`. Since this action mounts a volume over the current working directory by default, the current content of the working directory will be inaccessible afterwards.

With the default configuration, you won't gain any disk space. You can remove software that is unnecessary for your build job (see [this table](https://github.com/AdityaGarg8/maximize-build-space/blob/test-report/README.md) for examples and the expectable amount of space.

When removing software, consider that the removal of large amounts of files (which this is) can take minutes to complete. On the upside, you'll get around 51 GB of disk space available if you actually need it.

```yaml
name: My build action requiring more space
on: push

jobs:
  build:
    name: Build my artifact
    runs-on: ubuntu-latest
    steps:
      - name: Maximize build space
        uses: ublue-os/remove-unwanted-software@v7

      - name: Checkout
        uses: actions/checkout@v3

      - name: Build
        run: |
          echo "Free space:"
          df -h
```

## Inputs

All inputs are optional and default to the following, gaining about 7-8 GB additional space.

```yaml
  remove-dotnet:
    description: 'Removes .NET runtime and libraries. (frees ~2 GB)'
    required: false
    default: 'true'
  remove-android:
    description: 'Removes Android SDKs and Tools. (frees ~9 GB)'
    required: false
    default: 'true'
  remove-haskell:
    description: 'Removes GHC (Haskell) artifacts. (frees ~5.2 GB)'
    required: false
    default: 'true'
  remove-codeql:
    description: 'Removes CodeQL Action Bundles. (frees ~5.4 GB)'
    required: false
    default: 'false'
  remove-docker-images:
    description: 'Removes cached Docker images. (frees ~3.2 GB)'
    required: false
    default: 'false'
  remove-large-packages:
    description: 'Removes unwanted large Apt packages. (frees ~3.1 GB)'
    required: false
    default: 'false'
  remove-cached-tools:
    description: 'Removes cached tools used by setup actions by GitHub. (frees ~8.3 GB)'
    required: false
    default: 'false'
  remove-swapfile:
    description: 'Removes the Swapfile. (frees ~4 GB)'
    required: false
    default: 'false'
  verbose:
    description: 'Enables detailed logging of the action'
    required: false
    default: 'false' 
```

[easimon/maximize-build-space]: https://github.com/easimon/maximize-build-space
