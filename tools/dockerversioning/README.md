# Description

Versioning tools for Dockerfile source repos. Previously, these tools were in  [runtimes-common](https://github.com/GoogleCloudPlatform/runtimes-common/tree/b39744e5a8588beba847a271e379b864a6ac2939), but have been moved to [click-to-deploy](https://github.com/GoogleCloudPlatform/click-to-deploy) for further development.

- `dockerfiles` generates versionsed Dockerfiles from a common template.
- `cloudbuild` generates a configuration file to build these Dockerfiles using
  [Google Container Builder](https://cloud.google.com/container-builder/docs/).

# Installation

- Install [Bazel, version 0.19.2](https://bazel.build) as the build tool.

- Clone this repo:

``` shell
git clone https://github.com/GoogleCloudPlatform/click-to-deploy.git
cd click-to-deploy/tools
```

- Build:

``` shell
bazel run //:gazelle
bazel build dockerversioning/scripts/dockerfiles:dockerfiles
bazel build dockerversioning/scripts/cloudbuild:cloudbuild
```

- Set the path to the built scripts:

``` shell
BAZEL_ARCH=linux_amd64_stripped
export PATH=$PATH:$PWD/bazel-bin/dockerversioning/scripts/dockerfiles/${BAZEL_ARCH}/
export PATH=$PATH:$PWD/bazel-bin/dockerversioning/scripts/cloudbuild/${BAZEL_ARCH}/
```

# Build the image

Execute the script below from the `tools/` directory:

```shell
cd tools/
docker build -t dockertools -f dockertools.Dockerfile .
```

Add the alias to your terminal session:

```shell
alias c2d_tools='docker run \
  --rm \
  --workdir /mounted \
  --mount type=bind,source="$(pwd)",target=/mounted \
  --user $(id -u):$(id -g) \
  dockertools'
```

Test in a Docker folder:

```shell
# Generate the Cloud Build manifest
c2d_tools cloudbuild | tee cloudbuild.yaml

# Test dockerfiles binary
c2d_tools dockerfiles -verify_only
```

# Create `versions.yaml`

At root of the Dockerfile source repo, add a file called `versions.yaml`.
Follow the format defined in `versions.go`. See an example on
[github](https://github.com/GoogleCloudPlatform/mysql-docker).

Primary folders in the Dockerfile source repo:

- `templates` contains `Dockerfile.template`, which is a Go template for
  generating `Dockerfile`s.
- `tests` contains any tests that should be included in the generated cloud
  build configuration.
- Version folders as defined in `versions.yaml`. The `Dockerfile`s are
  generated into these folders. The folders should also contain all
  supporting files for each version, for example `docker-entrypoint.sh` files.

# Usage of `dockerfiles` command

``` shell
cd path/to/dockerfile/repo
dockerfiles
```

# Usage of `cloudbuild` command

``` shell
cd path/to/dockerfile/repo
cloudbuild > cloudbuild.yaml
```

You can use the generated `cloudbuild.yaml` file as followed:

``` shell
gcloud container builds submit --config=cloudbuild.yaml .
```
