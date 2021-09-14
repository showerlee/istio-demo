# istio-demo
A simple demo that helps you build micro service via istio framework in k8s.

## Context

### What is istio?

It is a completely open source service mesh that is connected to existing distributed applications as a transparent layer.

It is also a platform that can be integrated with any logging, telemetry and policy system. Istio's diverse features allow you to successfully and efficiently run a microservice architecture, and provide a unified method for protecting, connecting, and monitoring microservices.

![istio-architect](./docs/istio-architect.png)

### Why we choose istio?

- Traffic management
- Observability
- Security capabilities

More details: https://istio.io/latest/docs/

## How it works

We will create the following micrio services via GitOps to demonstrate how to achieve the traffic management, observability and security capabilities in `Istio Mesh`

![demo-frame](./docs/demo-frame.png)

- `Gateway` is a network ingress receives any vaild inbound traffic and forwards to internal services.

- [Httpbin](https://httpbin.org/) is a simple HTTP Request & Response Service.

- `Sleep` is a client that initiates a internal network request.

- `External` means any outbound traffic going outside `Istio Mesh`

### Prerequisite

[Setup-istio-in-docker-desktop](https://github.com/showerlee/k8s_tutorial/blob/master/manifests/istio/istioctl/README.md#setup-istio-in-docker-desktop)

### GitOps deployment via Flux

[Flux](https://fluxcd.io/docs/) helps you deploy istio resources via [GitOps](https://www.gitops.tech/#what-is-gitops) without any manual execution.

![flux-frame](./docs/flux-frame.png)

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
  git add demo/
  gcmsg "Add demo manifests"
  git push

  # Inject side car for namespace demo
  kubectl label namespace demo istio-injection=enabled --overwrite=true

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

### Canary release via Flagger

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
    --set slack.url=xxx \
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

  # Create latency metric for prometheus
  kubectl apply -f flagger/latency-template.yaml

  # Add host in local
  sudo echo "127.0.0.1 httpbin.example.com" /etc/hosts

  # Create canary analysis
  kubectl apply -f flagger/canary.yaml

  # Simulate the request for httpbin
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
  ...

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

  # Check canary result(refer to the canary release is switching sucessfully)
  kubectl get canary -n demo
  NAME                         STATUS      WEIGHT   LASTTRANSITIONTIME
  canary.flagger.app/httpbin   Succeeded   0        2021-09-07T16:31:31Z

  # Check grafana dashboard for canary
  kubectl port-forward -n istio-system svc/flagger-grafana 3000:80

  Visit localhost:3000
  ```


### Improve the flexibility of the system

![flexibility-process](./docs/flexibility-process.png)

```
# Get virtualservice
kubectl get virtualservice -n demo
NAME      GATEWAYS                                    HOSTS                               AGE
httpbin   ["public-gateway.demo.svc.cluster.local"]   ["httpbin.example.com"]             134m

Visit httpbin.example.com should display a httpbin demo frontend

# Create a new gateway for demo
kubectl apply -f istio/gateway.yaml

# Create a virtualservice for 1s timeout
kubectl apply -f istio/vs-timeout.yaml

# Simulate 2s delay for httpbin to expect a request timeout where the timeout threshold is 1s
curl http://httpbin.demo.example.com/delay/2
upstream request timeout%

# Create a VS for retry
kubectl apply -f istio/vs-retry.yaml

# Simulate a 500 request for httpbin to expect a retry action from logs
curl http://httpbin.demo.example.com/status/500

# Check `istio-proxy` container in httpbin whether a retry is triggered
kubectl logs -n demo httpbin-primary-77b74656cf-gzh7j -c istio-proxy -f

# Simulate a Circuit breaking for httpbin
kubectl apply -f istio/cb.yaml

# Setup loadtest via `fortio`
kubectl apply -f istio/fortio-deploy.yaml
FORTIO_POD=$(kubectl get pod -n demo | grep fortio | awk '{print $1}')

# Request 3 concurrencies and 30 times
kubectl exec -it -n demo "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 3 -qps 0 -n 30 -loglevel Warning http://httpbin.demo:8000/get
...
Code 200 : 14 (46.7 %)
Code 503 : 16 (53.3 %)
...
It implies 16(53.3% of) requests in 3 concurrencies and 30 times is blocked by Circuit breaking

# Check the fortio report
kubectl exec -n demo $FORTIO_POD -c istio-proxy -- pilot-agent request GET stats | grep httpbin.demo | grep pending
...
cluster.outbound|8000||httpbin.demo.svc.cluster.local.upstream_rq_pending_overflow: 16
cluster.outbound|8000||httpbin.demo.svc.cluster.local.upstream_rq_pending_total: 14

It gives the report that upstream_rq_pending_overflow is 16
```

### Setup secure policy

![auth-frame](./docs/auth-frame.png)

```
# Apply default policy(implicitly deny everything)
kubectl apply -f istio/ap-default.yaml

# Check whether request is deny
kubectl exec -it -n demo sleep-65c4679954-b955q -c sleep -- curl http://httpbin.demo:8000/get
RBAC: access denied

# Apply the policy where source namespace is demo
kubectl apply -f istio/ap-ns-demo.yaml

# Check if the accessiability is allowable where source namespace is demo and serviceaccount is sleep
kubectl exec -it -n demo sleep-65c4679954-b955q -c sleep -- curl http://httpbin.demo:8000/get
{"args":{},"headers":{"Accept":"*/*","Host":"httpbin.demo:8000","User-Agent":"curl/7.77.0","X-B3-Parentspanid":"e76a692f21e8c12e","X-B3-Sampled":"1","X-B3-Spanid":"a09b92f9128c4da3","X-B3-Traceid":"b804435ac4f79773e76a692f21e8c12e","X-Envoy-Attempt-Count":"1","X-Forwarded-Client-Cert":"By=spiffe://cluster.local/ns/demo/sa/httpbin;Hash=709cb97c88a409eee19553dc269d44849a4c31a59fb9114368b7f391fed95357;Subject=\"\";URI=spiffe://cluster.local/ns/demo/sa/sleep"},"origin":"127.0.0.6","url":"http://httpbin.demo:8000/get"}

kubectl exec -it sleep-854565cb79-5kczn -c sleep -- curl http://httpbin.demo:8000/get
RBAC: access denied%

# Apply the policy where source namespace is demo and destination operation is GET
kubectl apply -f istio/ap-ns-demo-get.yaml

# Check if the accessiability is allowable where source namespace is demo and serviceaccount is sleep and destination operation is GET
kubectl exec -it -n demo sleep-65c4679954-b955q -c sleep -- curl http://httpbin.demo:8000/get
{"args":{},"headers":{"Accept":"*/*","Host":"httpbin.demo:8000","User-Agent":"curl/7.77.0","X-B3-Parentspanid":"d97f06ddbe825330","X-B3-Sampled":"1","X-B3-Spanid":"e3aaefd18964854a","X-B3-Traceid":"2d0be3e03d5129f8d97f06ddbe825330","X-Envoy-Attempt-Count":"1","X-Forwarded-Client-Cert":"By=spiffe://cluster.local/ns/demo/sa/httpbin;Hash=709cb97c88a409eee19553dc269d44849a4c31a59fb9114368b7f391fed95357;Subject=\"\";URI=spiffe://cluster.local/ns/demo/sa/sleep"},"origin":"127.0.0.6","url":"http://httpbin.demo:8000/get"}

kubectl exec -it -n demo sleep-65c4679954-b955q -c sleep -- curl http://httpbin.demo:8000/headers
RBAC: access denied%

# Apply the policy where source namespace is demo and destination operation is GET and have proper headers
kubectl apply -f istio/ap-ns-demo-get-hds.yaml

# Check if the accessiability is allowable where source namespace is demo and serviceaccount is sleep and destination operation is GET and request.headers contains test string
kubectl exec -it -n demo sleep-65c4679954-b955q -c sleep -- curl http://httpbin.demo:8000/get
RBAC: access denied%

kubectl exec -it -n demo sleep-65c4679954-b955q -c sleep -- curl http://httpbin.demo:8000/get -H x-rfma-token:test1
{"args":{},"headers":{"Accept":"*/*","Host":"httpbin.demo:8000","User-Agent":"curl/7.77.0","X-B3-Parentspanid":"8a1b87146fc7c68e","X-B3-Sampled":"1","X-B3-Spanid":"8d68039464aef8d9","X-B3-Traceid":"2c0d9693b50fdf048a1b87146fc7c68e","X-Envoy-Attempt-Count":"1","X-Forwarded-Client-Cert":"By=spiffe://cluster.local/ns/demo/sa/httpbin;Hash=709cb97c88a409eee19553dc269d44849a4c31a59fb9114368b7f391fed95357;Subject=\"\";URI=spiffe://cluster.local/ns/demo/sa/sleep","X-Rfma-Token":"test1"},"origin":"127.0.0.6","url":"http://httpbin.demo:8000/get"}

```

### Collect metrics and monitor applications

```
# Visit prometheus
istioctl d prometheus

# Check envoy interface is workable
kubectl exec -it -n demo sleep-596dd95b6d-478sx -c sleep -- curl http://httpbin.demo:15090/stats/prometheus
...
envoy_cluster_assignment_stale{cluster_name="xds-grpc"} 0
...

# Visit grafana
istioctl d grafana
```

### Logging via ELK

![logging](./docs/logging.png)

```
# Create a new namespace for elk
kubectl create ns elk

# Setup ELK
kubectl apply -f elk/

# Check filebeat logs for validation
kubectl logs -n elk filebeat-8646b847b7-vzp42
...
2021-09-14T08:02:20.179Z	INFO	log/harvester.go:251	Harvester started for file: /var/log/containers/httpbin-5f54b6d56f-pf5m2_demo_httpbin-5f5d90d59e826546d3c7f61a7adbdbc3f3eaccc08e7c047d1f92bde7e8af043c.log
...

# Setup log index in kibana

```
