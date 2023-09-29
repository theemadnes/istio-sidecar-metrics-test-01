```
kubectl apply -f ClusterPodMonitoring.yaml

gcloud container clusters update autopilot-cluster-1 \
    --dataplane-v2-observability-mode=INTERNAL_VPC_LB --project mc-e2m-01

gcloud container clusters update autopilot-cluster-3 \
    --dataplane-v2-observability-mode=INTERNAL_VPC_LB --project mc-e2m-01 --region us-east4

kubectx autopilot-cluster-us-central1

gcloud container clusters update autopilot-cluster-us-central1 \
    --dataplane-v2-observability-mode=INTERNAL_VPC_LB --project gateway-multicluster-01 --region us-central1

kubectx autopilot-cluster-us-east4

kubectl apply -f ClusterPodMonitoring.yaml

gcloud container clusters update autopilot-cluster-us-east4 \
    --dataplane-v2-observability-mode=INTERNAL_VPC_LB --project gateway-multicluster-01 --region us-east4

brew install yq

while true; do grpcurl frontend.endpoints.gateway-multicluster-01.cloud.goog:443 whereami.Whereami.GetPayload | jq . && sleep 0.5; done 

kubectx gke_mc-e2m-01_us-central1_autopilot-cluster-1

mkdir -p relay-certs
kubectl -n kube-system get secret hubble-relay-client-certs \
    -o "jsonpath={.data['ca\.crt']}" | base64 -d >relay-certs/ca.crt
kubectl -n kube-system get secret hubble-relay-client-certs \
    -o "jsonpath={.data['tls\.crt']}" | base64 -d >relay-certs/client.crt
kubectl -n kube-system get secret hubble-relay-client-certs \
    -o "jsonpath={.data['tls\.key']}" | base64 -d >relay-certs/client.key

docker pull gcr.io/gke-release/cilium/hubble-cli@sha256:53e58ae42b2299949e8c2a8fedda0c142b72b7111e6f316d88788d4227ed4733

export RELAY_SERVICE_IP=`kubectl -n kube-system get svc hubble-ilb-svc \
    -o "jsonpath={.status.loadBalancer.ingress[0].ip}"`

docker run -it --rm \
    -v $PWD/relay-certs:/relay-certs:ro \
    -e HUBBLE_SERVER=tls://${RELAY_SERVICE_IP}:443 \
    -e HUBBLE_TLS_CLIENT_CERT_FILE=/relay-certs/client.crt \
    -e HUBBLE_TLS_CLIENT_KEY_FILE=/relay-certs/client.key \
    -e HUBBLE_TLS_CA_CERT_FILES=/relay-certs/ca.crt \
    -e HUBBLE_TLS_SERVER_NAME=relay.kube-system.svc.cluster.local \
    gcr.io/gke-release/cilium/hubble-cli@sha256:53e58ae42b2299949e8c2a8fedda0c142b72b7111e6f316d88788d4227ed4733 \
    status

docker run -it --rm \
    -v $PWD/relay-certs:/relay-certs:ro \
    -e HUBBLE_SERVER=tls://${RELAY_SERVICE_IP}:443 \
    -e HUBBLE_TLS_CLIENT_CERT_FILE=/relay-certs/client.crt \
    -e HUBBLE_TLS_CLIENT_KEY_FILE=/relay-certs/client.key \
    -e HUBBLE_TLS_CA_CERT_FILES=/relay-certs/ca.crt \
    -e HUBBLE_TLS_SERVER_NAME=relay.kube-system.svc.cluster.local \
    gcr.io/gke-release/cilium/hubble-cli@sha256:53e58ae42b2299949e8c2a8fedda0c142b72b7111e6f316d88788d4227ed4733 \
    observe

kubectl create namespace hubble-ui

kubectl -n kube-system get secrets hubble-relay-client-certs -oyaml  | \
    yq eval 'del(.metadata.namespace, .metadata.annotations, .metadata.uid, .metadata.creationTimestamp, .metadata.resourceVersion)' | \
    kubectl -n hubble-ui create -f -

kubectl -n kube-system get secrets hubble-relay-client-certs -oyaml  | \
    yq eval 'del(.metadata.namespace, .metadata.annotations, .metadata.uid, .metadata.creationTimestamp, .metadata.resourceVersion)' | \
    kubectl -n hubble-ui create -f -

kubectl apply -f hubble-ui-auto.yaml

kubectl -n hubble-ui port-forward service/hubble-ui 16100:80 --address='0.0.0.0'
```