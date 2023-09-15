```
# testing 
while true; do curl -s 34.30.135.45 | jq "." && sleep 0.25; done

# set up standalone prom proxy
kubectl create ns monitoring

gcloud config set project e2m-doc-01 \
&&
gcloud iam service-accounts create gmp-test-sa \
&&
gcloud iam service-accounts add-iam-policy-binding \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:e2m-doc-01.svc.id.goog[monitoring/default]" \
  gmp-test-sa@e2m-doc-01.iam.gserviceaccount.com \
&&
kubectl annotate serviceaccount \
  --namespace monitoring \
  default \
  iam.gke.io/gcp-service-account=gmp-test-sa@e2m-doc-01.iam.gserviceaccount.com


gcloud projects add-iam-policy-binding e2m-doc-01 \
  --member=serviceAccount:gmp-test-sa@e2m-doc-01.iam.gserviceaccount.com \
  --role=roles/monitoring.viewer

curl https://raw.githubusercontent.com/GoogleCloudPlatform/prometheus-engine/v0.7.0/examples/frontend.yaml |
sed 's/\$PROJECT_ID/e2m-doc-01/' |
kubectl apply -n monitoring -f -

# istio_io:service_client_sent_bytes_count{monitored_resource="k8s_pod"}
# istio_io:service_server_request_count{monitored_resource="k8s_container"} # this works 
# istio_io:service_server_connection_open_count{monitored_resource="k8s_container"}

# create test tcp workload 
kubectl create ns tcp-example
kubectl label namespace tcp-example istio-injection=enabled --overwrite

# set up grafana 

# set up helm
brew install helm

helm repo add kiali https://kiali.org/helm-charts

# install kiali
helm install \
    --set cr.create=true \
    --set cr.namespace=istio-system \
    --namespace kiali-operator \
    --create-namespace \
    kiali-operator \
    kiali/kiali-operator

kubectl create ns kiali

kubectl apply -f my-kiali-cr.yaml -n kiali

kubectl edit kiali kiali -n kiali

kubectl -n kiali create token kiali-service-account



```