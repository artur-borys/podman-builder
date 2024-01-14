# Podman Builder Image

This image is meant to be used with Gitea Actions when building and managing container images using buildah and podman.
I couldn't find any existing image from trusted entities, which has both podman and buildah setup properly for use in Gitea Actions.

The image is based on official podman and buildah images:

- https://github.com/containers/buildah/blob/main/contrib/buildahimage/Containerfile
- https://github.com/containers/podman/blob/main/contrib/podmanimage/stable/Containerfile

There are three flavors of the image, depending on the base image:

- `rocky` - based on `rocky:9` image. I recommend using this as a base image if you want to add more tools to it
- `rocky-minimal` - based on `rocky:9-minimal`. I recommend using this image for running the jobs requiring only nodejs, podman and buildah. Rocky minimal images contain `microdnf` instead of `dnf`, and you may encounter issues with that.
- `fedora` - based on `fedora:39`, same as the original podman and buildah images. Size of this image is a lot bigger than Rocky based images. It takes a long time to build it for `arm64` on QEMU, so I may decide to remove it or provide only `amd64` variant.

There are three tag variants:

- `latest-${variant}` - the latest build of given flavor
- `${github.sha}-${variant}` - specific git commit hash build
- `${version}-${variant}` - specific version (i.e `v1.0.0`) build

`latest` is not recommended. Version-based tag is better, but it's always the most secure to use the commit hash tag.

## How to use it

If you're running `act_runner` via Docker (and I assume the same applies for Kubernetes, but I haven't tried it yet), it's important to:

- set `--security-opt seccomp=unrestricted`
- bind `fuse` device via `--device /dev/fuse`
- when running on Fedora/RHEL/CentOS/Rocky, or any OS with SELinux, you may also need to add `--security-opt label=disable`

When act_runner spawns a container where a job will run, it also has to pass the same options as above, so now you have to make a choice:

- add the options to act_runner config file, so that they will be automatically added to all containers
- specify the options in `jobs.<JOB>.container.options`

The first option might be less secure, but you can always have two runners - one with options in config file - for podman jobs, one without those options - for other jobs.
Then you can assing jobs requiring podman or buildah to the first runner.

If you want to configure your runner to use this `podman-runner` image, the `config.yaml` must contain this section (adapt it to your needs):

```yaml
runner:
  labels:
    - rocky-minimal:docker://aborys/podman-builder:latest-rocky-minimal
    - rocky:docker://aborys/podman-builder:latest-rocky
    - fedora:docker://aborys/podman-builder:latest-fedora

container:
  options: --security-opt label=disable --security-opt seccomp=unconfined --device /dev/fuse:rw
```

Example:

```bash
docker run --rm -it \
  --security-opt seccomp=unconfined \
  -v $PWD/config.yaml:/config.yaml
  --device /dev/fuse \
  -e GITEA_INSTANCE_URL='<<YOUR_GITEA_INSTANCE>>' \
  -e CONFIG_FILE=/config.yaml \
  -e GITEA_RUNNER_REGISTRATION_TOKEN='<<YOUR_REGISTRATION_TOKEN>>' \
  --name runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitea/act_runner:nightly
```

Alternatively, see [Docker Compose example](./example)

You can then run jobs with this image:

```yaml
jobs:
  <<JOB_NAME>>:
    runs-on: rocky-minimal
    # Add this if you didn't set container.options in config.yaml
    # container:
    #   options: --security-opt label=disable --security-opt seccomp=unconfined --device /dev/fuse:rw
    ...
```

You can see a full working workflow example in [.gitea/workflows/build.yaml](.gitea/workflows/build.yaml)

## Security

I know that the built images have some high level vulnerabilities and I plan to fix them. At a first glance most of them look like issues with Node.js, which is unfortunately required by a lot of actions.

The container itself runs as a `build` user by default.
