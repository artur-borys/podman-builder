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

## How to use it

If you're running `act_runner` via Docker (and I assume the same applies for Kubernetes, but I haven't tried it yet), it's important to:

- set `--security-opt seccomp=unrestricted`
- bind `fuse` device via `--device /dev/fuse`
- when running on Fedora/RHEL/CentOS/Rocky, or any OS with SELinux, you may also need to add `--security-opt label=disable`

Example:

```bash
docker run --rm -it \
  --security-opt seccomp=unconfined \
  --device /dev/fuse \
  -e GITEA_INSTANCE_URL='<<YOUR_GITEA_INSTANCE>>' \
  -e GITEA_RUNNER_REGISTRATION_TOKEN='<<YOUR_REGISTRATION_TOKEN>>' \
  --name runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitea/act_runner:nightly
```

Then, you have to override the container image used by your jobs (I have yet to learn about runner labels):

```yaml
jobs:
  <<JOB_NAME>>:
    container:
      image: aborys/podman-builder:latest-rocky-minimal
      options: --security-opt label=disable --security-opt seccomp=unconfined --device /dev/fuse:rw
    ...
```

You can see a full working example in [.gitea/workflows/build.yaml](.gitea/workflows/build.yaml)

## Security

I know that the built images have some high level vulnerabilities and I plan to fix them. At a first glance most of them look like issues with Node.js, which is unfortunately required by a lot of actions.

The container itself runs as a `build` user by default.

I've yet to master Github Actions in itself, so I'm not entirely sure how to create git tags and how to then use them for building images, so currently there are only two types of tags provided:

- `latest-<flavor>` - the latest built version
- `<git.sha>-<flavor>` - static image tag with the git commit SHA checksum, which can be used if you have to lock a specific version
