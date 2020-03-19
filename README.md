# Cuttlefacts infrastructure (with Argo)

This is a companion to
[cuttlefacts/cuttlefacts](https://github.com/cuttlefacts/cuttlefacts/),
using [Argo kit](https://github.com/argoproj/) instead of Tekton to
drive application delivery
pipelines. [Flux](https://github.com/fluxcd/flux) is still used to
bootstrap the infrastructure.

## How to use this repository

From a "bare" cluster (Docker for Desktop will do):

    kubectl apply -f bootstrap/

You will probably want to plat around with the definitions and so
on. Fork this repo, then update the git URL in `bootstrap/flux.yaml`,
and go from there.

For image builds to work, you'll need to set up a secret with write
access to a Docker image repository -- see
[local/README.md](./local/README.md).
