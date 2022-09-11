# How to set up local cluster for development

## Setup local k8s cluster with rancher desktop and K3s

## Use Lens

Exec into worker node and `mkdir /mnt/chartmuseum` for `chartmuseum` PV

Obtain node labels and set `nodeAffinity` for `infrastructure/chartmuseum/persistent-volume.yaml`

## Setup Flux

Flux gitops stack and Kpack will be installed automatically.

Install flux2 cli
```bash
brew install fluxcd/tap/flux
```

Bootstrap flux for local cluster:
```bash
export GITHUB_TOKEN="X"
export GITHUB_USER="X"

flux bootstrap github \
  --components-extra=image-reflector-controller,image-automation-controller \
  --owner=$GITHUB_USER \
  --repository=flux-gitops \
  --path=clusters/local \
  --read-write-key \
  --personal
```

Flux will generate ssh keypair, create Secret and add public part to deploy keys of you github repo using your personal access token.
We use `--read-write-key` option to allow flux (ImageAutomation controller) to change resources in our flux repo.

## Setup Kpack

Install Kpack cli. See docs for cli commands https://github.com/vmware-tanzu/kpack-cli/blob/main/docs/kp.md
```bash
brew tap vmware-tanzu/kpack-cli
brew install kp
```

Create ImagePull Secret for dockerhub registry
```bash
export REGISTRY_USER="X"
export REGISTRY_PASS="X"
export REGISTRY_URL=https://index.docker.io/v1/

kubectl create secret docker-registry dockerhub-credentials \
--docker-username=$REGISTRY_USER \
--docker-password=$REGISTRY_PASS \
--docker-server=$REGISTRY_URL \
--namespace build
```

## Helm chart

Build and push helm chart to chartmuseum:
```bash
git clone git@github.com:antonioua/gitops-demo-app.git
cd gitops-demo-app
helm package ./helm-charts/gitops-demo-app/ --version 1.0.0
kubectl port-forward -n infra svc/ac-chartmuseum 8080:8080
curl --data-binary "@gitops-demo-app-1.0.0.tgz" http://localhost:8080/api/charts
```

## How to maintain Flux

Trigger reconcile of flux controllers manually:
```bash
flux reconcile kustomization flux-system --with-source # Tell Flux to pull and apply the changes
flux reconcile source git flux-system
flux reconcile kustomization apps
flux reconcile kustomization infrastructure # if you just want to update infra
flux reconcile image update gitops-demo-app-rollout -n flux-system # run image automation
```

Check controllers statuses
```bash
flux get sources all
flux get sources git # kubectl get GitRepository -A
flux get kustomization # kubectl get kustomizations.kustomize.toolkit.fluxcd.io -A
flux get image all -A # get policy repo and automation statuses
flux get image repository -A # kubectl get ImageRepository -A
flux get image policy -A # kubectl get ImagePolicy -A
flux get images update -A # kubectl get ImageUpdateAutomation -A
flux get helmreleases -A # kubectl get HelmRelease -A
```

Check Kpack controllers statuses
```bash
kp clusterstore status default
kp clusterstack status base
kp builder status my-go-builder -n build
kp image status gitops-demo-app -n build
kp build logs gitops-demo-app -n build
```

Observe API resource fields
```bash
kubectl explain ImagePolicy.spec.policy
```

Other flux commands:
```bash
flux suspend image repository gitops-demo-app
flux uninstall --namespace=flux-system
```

## Todo
- Tag images in kpack like ${GIT_BRANCH}-${GIT_SHA:0:7}-$(date +%s) # main-2d3fcbd-1611906956
- Optimize directory structure as per example: https://github.com/fluxcd/flux2-kustomize-helm-example