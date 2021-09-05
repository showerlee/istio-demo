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

  # Check all resources in demo namespace
  kubectl get all -n demo
  NAME                           READY   STATUS    RESTARTS   AGE
  pod/httpbin-74fb669cc6-8kbzh   1/1     Running   0          9m1s
  pod/sleep-854565cb79-2vdrl     1/1     Running   0          53m

  NAME              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
  service/httpbin   ClusterIP   10.101.24.23   <none>        8000/TCP   9m1s
  service/sleep     ClusterIP   10.111.29.70   <none>        80/TCP     62m

  NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
  deployment.apps/httpbin   1/1     1            1           9m1s
  deployment.apps/sleep     1/1     1            1           53m

  NAME                                 DESIRED   CURRENT   READY   AGE
  replicaset.apps/httpbin-74fb669cc6   1         1         1       9m1s
  replicaset.apps/sleep-854565cb79     1         1         1       53m

  # Check the connection between sleep and httpbin
  kubectl exec -it -n demo sleep-854565cb79-2vdrl -c sleep -- curl http://httpbin.demo:8000/ip
  {
    "origin": "10.1.1.71"
  }
  ```
