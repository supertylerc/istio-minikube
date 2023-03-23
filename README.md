```bash
# Create a k8s cluster with enough RAM and CPU
minikube start --memory=15425 --cpus=4

# Install the k8s Gateway API CRDs for new hotness
kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd" | kubectl apply -f -

# Get the latest Istio release locally
curl -L https://istio.io/downloadIstio | sh -

# Add the istioctl binary to your bath
cd istio-1*
export PATH=$PWD/bin:$PATH

# Install istio
istioctl install --set profile=demo -y

# Install the demo application
cat <<EOF > kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- samples/bookinfo/platform/kube/bookinfo.yaml
patches:
- path: patch.yaml
  target:
    kind: Deployment
EOF

cat <<EOF > patch.yaml
kind: Deployment
metadata:
  name: istio-inject
spec:
  template:
    metadata:
      labels:
        sidecar.istio.io/inject: "true"
EOF

kubectl kustomize . | kubectl apply -f -

# Install the Gateway
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml

# Install the addons
kubectl apply -f samples/addons

# Start a tunnel in another terminal
minikube tunnel

export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT


# In a separate terminal, generate traffic
while :; do ; do curl -s -o /dev/null "http://$GATEWAY_URL/productpage"; sleep 0.1; done


echo "Visit the product page: http://$GATEWAY_URL/productpage"

# In another terminal, open Kiali
istioctl dashboard kiali
istioctl dashboard jaeger
```

