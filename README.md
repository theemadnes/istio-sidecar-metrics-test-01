# istio-sidecar-metrics-test-01
using proxyConfig to expand proxy metrics collection

### testing

```
# basic calls 
grpcurl frontend.endpoints.gateway-multicluster-01.cloud.goog:443 whereami.Whereami.GetPayload | jq .
# or
while true; do grpcurl frontend.endpoints.gateway-multicluster-01.cloud.goog:443 whereami.Whereami.GetPayload | jq .; done

# example pilot-agent command
kubectl exec whereami-grpc-frontend-5db94bfbc8-ggpnl -n whereami-grpc-frontend -c istio-proxy -- pilot-agent request GET stats


```

instead, going to modify the `deployment` spec for `whereami-grpc-frontend` to include a test annotation to include all `http` metrics:

```
proxy.istio.io/config: |-
          proxyStatsMatcher:
            inclusionRegexps:
            - ".*http.*"
```

get a pod IP to test prom endpoint 

```
export POD_IP=${kubectl get pod -n whereami-grpc-frontend -o jsonpath='{.items[0].status.podIP}'}


kubectl exec --stdin --tty -n whereami-grpc-frontend whereami-grpc-frontend-5db94bfbc8-ggpnl -- /bin/sh
curl http://10.127.0.151:15020/stats/prometheus -s | grep "cluster.xds-grpc.upstream_cx_length_ms"
```

set up gmp KSA
```
gcloud config set project gateway-multicluster-01 \
&&
gcloud iam service-accounts create gmp-test-sa \
&&
gcloud iam service-accounts add-iam-policy-binding \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:gateway-multicluster-01.svc.id.goog[NAMESPACE_NAME/default]" \
  gmp-test-sa@gateway-multicluster-01.iam.gserviceaccount.com \
&&
kubectl annotate serviceaccount \
  --namespace NAMESPACE_NAME \
  default \
  iam.gke.io/gcp-service-account=gmp-test-sa@gateway-multicluster-01.iam.gserviceaccount.com
```