# istio-demo

![istio-frame](./docs/istio-frame.png)
![flux-frame](./docs/flux-frame.png)

## Prerequisite
https://github.com/showerlee/k8s_tutorial/blob/master/manifests/istio/istioctl/README.md#setup-istio-in-docker-desktop

## GitOps deployment via Flux

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
  pod/httpbin-74fb669cc6-8kbzh   2/2     Running   0          9m1s
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

## Canary release via Flagger

![canary-process](./docs/canary-process.png)
![flagger-process](./docs/flagger-process.png)

- Flagger

  ```
  # Install flagger CRD
  kubectl apply -f flagger/crd.yaml

  # Check flagger CRD
  kubectl get crd | grep flagger

  # Add Flagger helm repo
  helm repo add flagger https://flagger.app

  # Deploy flagger with istio
  helm upgrade -i flagger flagger/flagger \
    --namespace=istio-system \
    --set crd.create=false \
    --set meshProvider=istio \
    --set metricsServer=http://prometheus.istio-system:9090

  # Create a test Slack workspace and Flagger APP for webhook
  https://api.slack.com/messaging/webhooks

  # Add slack webhook for flagger
  helm upgrade -i flagger flagger/flagger \
    --namespace=istio-system \
    --set crd.create=false \
    --set slack.url=https://hooks.slack.com/services/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 \
    --set slack.channel=istio-demo \
    --set slack.user=Flagger

  # Add grafana for flagger dashboard
  helm upgrade -i flagger-grafana flagger/grafana \
    --namespace=istio-system \
    --set url=http://prometheus.istio-system:9090 \
    --set user=admin \
    --set password=admin

  # Check the outcome
  kubectl get pods -n istio-system
  NAME                                    READY   STATUS    RESTARTS   AGE
  flagger-6d6fcd75cc-wvktw                1/1     Running   0          4m54s
  flagger-grafana-77b8c8df65-9gbkh        1/1     Running   0          83s
  grafana-5b865d9cfb-vz2tn                1/1     Running   0          2d23h
  istio-egressgateway-5dcd5999f4-qffwh    1/1     Running   0          2d23h
  istio-ingressgateway-7d754985d5-dcgwx   1/1     Running   0          2d23h
  istiod-66755bc76c-ksgjq                 1/1     Running   0          2d23h
  jaeger-99b4cd9bb-bs8tz                  1/1     Running   0          2d23h
  kiali-6f5f6d4bbd-fptkf                  1/1     Running   0          2d23h
  prometheus-5b96c5d456-mw957             3/3     Running   0          2d23h

  # Create istio ingress gateway
  kubectl apply -f flagger/istio-gateway.yaml

  # Deploy flagger loadtester in demo namespace
  https://github.com/fluxcd/flagger/tree/main/kustomize/tester

  kubectl apply -f flagger/tester-deployment.yaml -n demo
  kubectl apply -f flagger/tester-svc.yaml -n demo

  # Create hpa for httpbin
  kubectl apply -f flagger/hpa.yaml

  # Create canary analysis
  kubectl apply -f flagger/canary.yaml

  # Check canary analysis
  kubectl get canary -n demo
  NAME      STATUS         WEIGHT   LASTTRANSITIONTIME
  httpbin   Initializing   0        2021-09-07T14:21:52Z

  # Check all resources that canary done for demo
  kubectl get all -n demo

  ```
