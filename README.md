# istio-demo

![istio-frame](./docs/istio-frame.png)
![flux-frame](./docs/flux-frame.png)
## Init env

- [Flux](https://fluxcd.io/docs/components/source/)

  ```
  # Install flux client
  brew install fluxcd/tap/flux

  # Check k8s cluster
  flux check --pre

  # Export your credentials
  export GITHUB_TOKEN=<your-token>
  export GITHUB_USER=<your-username>

  # Setup Flux onto k8s cluster
  flux bootstrap github \
    --owner=$GITHUB_USER \
    --repository=istio-demo \
    --branch=main \
    --path=./flux \
    --personal
  ```

- Demo

  ```
  # Create namespace for demo
  kubectl create ns demo

  # Commit demo manifests into repo so that flux could manage the deployment.
  gaa
  gcmsg "Add demo manifests"
  gp

  # Export the manifest of demo source git for flux
  flux create source git demo \
    --url=https://github.com/showerlee/istio-demo \
    --branch=main \
    --interval=30s \
    --export > ./flux/demo-source.yaml
  
  # Commit and push demo-source.yaml to flux base dir:
  git add -A && git commit -m "Add demo GitRepository"
  git push

  # Export the manifest of demo deployment for flux
  flux create kustomization demo \
    --source=demo \
    --path="./demo" \
    --prune=true \
    --validation=client \
    --interval=5m \
    --export > ./flux/demo-kustomization.yaml

  # Commit and push demo-source.yaml to flux base dir:
  git add -A && git commit -m "Add demo Kustomization"
  git push
  ```
