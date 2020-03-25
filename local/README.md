# Local configuration needed

Naturally, there will always be configuration particular to the
environment. In this case, it's secrets.

I have not bothered setting up any secrets management, I just rely on
creating a secret at some point after the cluster has been
bootstrapped.

There are a few secrets needed.

## Setting up an image push secret

I use the GitHub package repository for storing images (the image refs
are mentioned in the platform layer).

To create an image push secret -- that is, one that has the [ability
to push to an image repository][push-perms] -- I did the following:

 - I created a machine account and gave it 'Write' role in this
   (cuttlefacts-argo) repository;
 - I created a personal access token for that account, with
   `read:packages` and `write:packages` permissions (and no `repo`
   permissions), then after copying it to the clipboard,

```bash
# the name 'cuttlefacts-push-secret' matches the secret
# mounted as a volume in platform/cuttlefacts-build.yaml
kubectl create secret docker-registry --dry-run -o yaml \
  -n platform cuttlefacts-push-secret \
  --docker-server=https://docker.pkg.github.com \
  --docker-username=squaremobot \
  --docker-password=$(pbpaste) > secret.yaml
```

This makes a secret with an entry `.dockerconfigjson`, but I need
`config.json` so that it can be used when mounted into a
container. Either edit it to copy the `.dockerconfigjson` to
`config.json` (as well), or use `./bin/dockerconfigjson.jk`, and
apply.

[push-perms]: https://help.github.com/en/packages/publishing-and-managing-packages/about-github-packages#about-tokens

## Setting up an image **pull** secret

The app configuration will need an `imagePullSecret`, if you are using
a registry that needs authentication (GitHub packages does).

I just used the account token from above, but put it in the
`cuttlefacts` namespace and named it `pull-secret`:

```bash
# the name 'pull-secret' matches the imagePullSecret entry
# in the deployment
kubectl create secret docker-registry \
  -n cuttlefacts pull-secret \
  --docker-server=https://docker.pkg.github.com \
  --docker-username=squaremobot --docker-password=$(pbpaste)
```

## Setting up a webhook admin secret

This is required (evidently) by the Argo Triggers event source, for it
to set up the webhook for you. I used the machine account from before,
gave it `Admin` access to the cuttlefacts-app project, and created
another personal access token with `admin:repo_hook` permission. Then,

```
# the name 'cuttlefacts-admin-secret' matches the secretRef
# in platform/cuttlefacts-events.yaml.
kubectl create secret generic -n platform cuttlefacts-admin-secret \
  --from-literal=token=$(pbpaste)
```

## Setting up a shared webhook secret

GitHub signs webhook requests with a shared secret so the receiver can
verify the payload. To create a shared secret:

```
KEY=$(ruby -rsecurerandom -e 'print SecureRandom.hex(20)')
# the name 'cuttlefacts-webhook-secret' matches the webhookSecret
# in platform/cuttlefacts-events.yaml
kubectl -n platform create secret generic cuttlefacts-webhook-secret --from-literal=key=$KEY
```

## Ngrok

I use [ngrok](https://ngrok.io) to tunnel webhooks though to the
cluster. A deployment to run it, and point it at the Argo Events
gateway, is given in ngrok.yaml -- but see below for caveats.

Since the GitHub event source (in
[platform/cuttlefacts-events.yaml][]) wants to set the webhook up in
GitHub itself, you have to supply an endpoint. So that I don't have to
keep changing the hostname -- ngrok will create a random subdomain for
its tunnel, every time it starts up, unless otherwise told -- I have
used a subdomain.

To use ngrok in the same way, you need a paid account :-( But you may
be able to get away with fetching the domain name each time ngrok is
restarted (it'll only hold a tunnel open for a few hours, so you need
to restart it occasionally). Remove the `-subdomain` argument from the
container args, or change it if you have a paid account.

Either way, to see the ngrok console, port-forward to it. This is
really handy for debugging webhooks.

    kubectl -n platform port-forward deploy/ngrok 4040 &

.. then open `http://localhost:4040/` in a web browser.
