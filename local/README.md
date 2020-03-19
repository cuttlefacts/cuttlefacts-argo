# Local configuration needed

Naturally, there will always be configuration particular to the
environment. In this case, it's secrets.

I have not bothered setting up any secrets management, I just rely on
creating a secret at some point after the cluster has been
bootstrapped.

## Setting up an image push secret

I use the GitHub package repository for storing images (the image refs
are mentioned in the platform layer).

To create an image push secret -- that is, one that has the ability to
push to an image repository -- I did the following:

 - I created a personal access token in the settings page on GitHub,
   then after copying it to the clipboard,

```
# the name 'cuttlefacts-push-secret' matches the imagePullSecret
# in policy/triggers-serviceaccount.yaml
kubectl create secret docker-registry --dry-run -o yaml \
  -n platform cuttlefacts-push-secret \
  --docker-server=https://docker.pkg.github.com \
  --docker-username=squaremo \
  --docker-password=$(pbpaste) > secret.yaml
```

This makes a secret with an entry `.dockerconfigjson`, but I need
`config.json` so that it can be used when mounted into a
container. Either edit it so the data entry has the key `config.json`,
or use `./bin/dockerconfigjson.jk`, and apply.

## Setting up a shared webhook secret

GitHub signs webhook requests with a shared secret so the receiver can
verify the payload. To create a shared secret:

```
KEY=$(ruby -rsecurerandom -e 'print SecureRandom.hex(20)')
# the name 'cuttlefacts-webhook-secret' matches the webhookSecret
# in platform/cuttlefacts-events.yaml
kubectl -n platform create secret generic cuttlefacts-webhook-secret --from-literal=key=$KEY
```
