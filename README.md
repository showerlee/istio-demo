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
  NAME                                      READY   STATUS    RESTARTS   AGE
  pod/flagger-loadtester-57856ccd69-6bzk9   1/1     Running   0          32m
  pod/httpbin-primary-fcbdd7f6b-4ffz2       2/2     Running   0          52s
  pod/httpbin-primary-fcbdd7f6b-qmchh       2/2     Running   0          8m22s
  pod/sleep-854565cb79-2vdrl                1/1     Running   0          2d10h

  NAME                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
  service/flagger-loadtester   ClusterIP   10.104.130.51    <none>        80/TCP     32m
  service/httpbin              ClusterIP   10.101.24.23     <none>        8000/TCP   2d10h
  service/httpbin-canary       ClusterIP   10.102.6.228     <none>        8000/TCP   8m22s
  service/httpbin-primary      ClusterIP   10.107.153.184   <none>        8000/TCP   8m22s
  service/sleep                ClusterIP   10.111.29.70     <none>        80/TCP     2d10h

  NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
  deployment.apps/flagger-loadtester   1/1     1            1           32m
  deployment.apps/httpbin              0/0     0            0           2d10h
  deployment.apps/httpbin-primary      2/2     2            2           8m22s
  deployment.apps/sleep                1/1     1            1           2d10h

  NAME                                            DESIRED   CURRENT   READY   AGE
  replicaset.apps/flagger-loadtester-57856ccd69   1         1         1       32m
  replicaset.apps/httpbin-74fb669cc6              0         0         0       2d10h
  replicaset.apps/httpbin-primary-fcbdd7f6b       2         2         2       8m22s
  replicaset.apps/sleep-854565cb79                1         1         1       2d10h

  NAME                                                  REFERENCE                    TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
  horizontalpodautoscaler.autoscaling/httpbin           Deployment/httpbin           <unknown>/99%   2         4         0          2m20s
  horizontalpodautoscaler.autoscaling/httpbin-primary   Deployment/httpbin-primary   <unknown>/99%   2         4         1          112s

  NAME                         STATUS        WEIGHT   LASTTRANSITIONTIME
  canary.flagger.app/httpbin   Initialized   0        2021-09-07T14:48:38Z

  # Release httpbin test version for canary deployment
  Change the httpbin image version to test in `demo/httpbin.yaml` and commit the change for flux deployment

  # Check canary events
  kubectl describe canary -n demo
  Events:
  Type     Reason  Age                   From     Message
  ----     ------  ----                  ----     -------
  Normal   Synced  113s (x2 over 20m)  flagger  New revision detected! Scaling up httpbin.demo
  Warning  Synced  83s                 flagger  canary deployment httpbin.demo not ready: waiting for rollout to finish: 1 old replicas are pending termination
  Normal   Synced  53s (x2 over 19m)   flagger  Starting canary analysis for httpbin.demo
  Normal   Synced  106s (x4 over 37m)   flagger  Advance httpbin.demo canary weight 20

  # Check if virtual service starting switching traffic
  kubectl describe vs httpbin -n demo
  ...
    Http:
    Route:
      Destination:
        Host:  httpbin-primary
      Weight:  80
      Destination:
        Host:  httpbin-canary
      Weight:  20
  ...

  # Check the connection
  kubectl exec -it -n demo sleep-854565cb79-2vdrl -c sleep sh
  while [ 1 ]; do curl http://httpbin.demo:8000/headers;sleep 2s; done
  {
    "headers": {
      "Accept": "*/*",
      "Host": "httpbin.demo:8000",
      "User-Agent": "curl/7.77.0",
      "X-B3-Sampled": "0",
      "X-B3-Spanid": "c79b98a968ef9954",
      "X-B3-Traceid": "281e285c1e9c5f49c79b98a968ef9954"
    }
  }
  {
    "headers": {
      "Accept": "*/*",
      "Host": "httpbin.demo:8000",
      "User-Agent": "curl/7.77.0",
      "X-B3-Sampled": "0",
      "X-B3-Spanid": "685e339a26b0b414",
      "X-B3-Traceid": "3f32d3f1991e298f685e339a26b0b414"
    }
  }

  ```
