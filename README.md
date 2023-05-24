# e2m-gateway-api-01
refreshing E2M with Gateway API and integrating [Certificate Manager](https://cloud.google.com/certificate-manager/docs/overview)

### set environment vars
```
export PROJECT=am-arg-01
export CLUSTER_NAME=edge-to-mesh
export REGION=us-central1
export RELEASE_CHANNEL=rapid
export GKE_URI=https://container.googleapis.com/v1/projects/${PROJECT}/locations/${REGION}/clusters/${CLUSTER_NAME}
export PROJECT_NUMBER=$(gcloud projects list --filter=${PROJECT} --format="value(PROJECT_NUMBER)")
```
### enable APIs
```
gcloud services enable container.googleapis.com --project=${PROJECT}
gcloud services enable mesh.googleapis.com --project=${PROJECT}
gcloud services enable certificatemanager.googleapis.com --project=${PROJECT}
```

### create cluster and enable mesh
```
gcloud container --project ${PROJECT} clusters create-auto ${CLUSTER_NAME} --region ${REGION} --release-channel ${RELEASE_CHANNEL}

gcloud container fleet mesh enable --project=${PROJECT}

gcloud container fleet memberships register ${CLUSTER_NAME} \
  --gke-uri=${GKE_URI} \
  --project ${PROJECT}

# verify membership
gcloud container fleet memberships list --project ${PROJECT}

gcloud container clusters update ${CLUSTER_NAME} --project ${PROJECT} \
  --region ${REGION} --update-labels mesh_id=proj-${PROJECT_NUMBER}

# manually updating cluster to also include Gateway API for now
gcloud container clusters update ${CLUSTER_NAME} --project ${PROJECT} \
  --region ${REGION} --gateway-api=standard

# verify gateway classes
kubectl get gatewayclass

gcloud container fleet mesh update \
    --management automatic \
    --memberships ${CLUSTER_NAME} \
    --project ${PROJECT}

gcloud container fleet mesh describe --project ${PROJECT}
```

### Create ingress gateway namespace
```
kubectl create namespace asm-ingress
kubectl label namespace asm-ingress istio-injection=enabled
```

### Create self-signed certificate for ASM/Istio Gateway
```
openssl req -new -newkey rsa:4096 -days 365 -nodes -x509 \
 -subj "/CN=frontend.endpoints.${PROJECT}.cloud.goog/O=Edge2Mesh Inc" \
 -keyout frontend.endpoints.${PROJECT}.cloud.goog.key \
 -out frontend.endpoints.${PROJECT}.cloud.goog.crt

kubectl -n asm-ingress create secret tls edge2mesh-credential \
 --key=frontend.endpoints.${PROJECT}.cloud.goog.key \
 --cert=frontend.endpoints.${PROJECT}.cloud.goog.crt
```

### Create the asm-ingress NS and deployment, service, SA, role, rolebinding, and Gateway object
```
mkdir -p asm-ig/base

cat <<EOF > asm-ig/base/kustomization.yaml
resources:
  - github.com/GoogleCloudPlatform/anthos-service-mesh-samples/docs/ingress-gateway-asm-manifests/base
EOF

mkdir asm-ig/variant

cat <<EOF > asm-ig/variant/role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: asm-ingressgateway
  namespace: asm-ingress
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
EOF

cat <<EOF > asm-ig/variant/rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: asm-ingressgateway
  namespace: asm-ingress
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: asm-ingressgateway
subjects:
  - kind: ServiceAccount
    name: asm-ingressgateway
EOF

cat <<EOF > asm-ig/variant/service-proto-type.yaml 
apiVersion: v1
kind: Service
metadata:
  name: asm-ingressgateway
spec:
  ports:
  - name: status-port
    port: 15021
    protocol: TCP
    targetPort: 15021
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
    appProtocol: HTTP2
  type: ClusterIP
EOF

cat <<EOF > asm-ig/variant/gateway.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: asm-ingressgateway
spec:
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "*" # IMPORTANT: Must use wildcard here when using SSL, see note below
    tls:
      mode: SIMPLE
      credentialName: edge2mesh-credential
EOF

cat <<EOF > asm-ig/variant/kustomization.yaml 
namespace: asm-ingress
resources:
- ../base
- role.yaml
- rolebinding.yaml
patches:
- path: service-proto-type.yaml
  target:
    kind: Service
- path: gateway.yaml
  target:
    kind: Gateway
EOF

# apply
kubectl apply -k asm-ig/variant
```
**NOTE:** if you see an error then repeat the `kubectl apply` above. Warnings can be ignored

### create cloud armor security policy and reference via GCP backend policy
```
# https://cloud.google.com/kubernetes-engine/docs/how-to/configure-gateway-resources#configure_cloud_armor
gcloud compute security-policies create edge-fw-policy \
    --project=${PROJECT} --description "Block XSS attacks"

gcloud compute security-policies rules create 1000 \
    --security-policy edge-fw-policy \
    --expression "evaluatePreconfiguredExpr('xss-stable')" \
    --action "deny-403" \
    --project=${PROJECT} \
    --description "XSS attack filtering" 

cat <<EOF > cloud-armor-backendpolicy.yaml
apiVersion: networking.gke.io/v1
kind: GCPBackendPolicy
metadata:
  name: cloud-armor-backendpolicy
  namespace: asm-ingress
spec:
  default:
    securityPolicy: edge-fw-policy
  targetRef:
    group: ""
    kind: Service
    name: asm-ingressgateway
EOF

# apply backend policy
kubectl apply -f cloud-armor-backendpolicy.yaml
```

### create ingress gateway health check policy
```
# see https://cloud.google.com/kubernetes-engine/docs/how-to/configure-gateway-resources#configure_health_check
cat <<EOF > ingress-gateway-healthcheck.yaml
apiVersion: networking.gke.io/v1
kind: HealthCheckPolicy
metadata:
  name: ingress-gateway-healthcheck
  namespace: asm-ingress
spec:
  default:
    checkIntervalSec: 20
    timeoutSec: 5
    #healthyThreshold: HEALTHY_THRESHOLD
    #unhealthyThreshold: UNHEALTHY_THRESHOLD
    logConfig:
      enabled: True
    config:
      type: HTTP
      httpHealthCheck:
        #portSpecification: USE_NAMED_PORT
        port: 15021
        portName: status-port
        #host: HOST
        requestPath: /healthz/ready
        #response: RESPONSE
        #proxyHeader: PROXY_HEADER
    #requestPath: /healthz/ready
    #port: 15021
  targetRef:
    group: ""
    kind: Service
    name: asm-ingressgateway
EOF

# apply the healthcheck
kubectl apply -f ingress-gateway-healthcheck.yaml
```

### Configure IP addressing and DNS
```
gcloud --project=${PROJECT} compute addresses create ingress-ip --global

export GCLB_IP=$(gcloud --project=${PROJECT} compute addresses describe ingress-ip --global --format "value(address)")
echo ${GCLB_IP}

cat <<EOF > dns-spec.yaml
swagger: "2.0"
info:
  description: "Cloud Endpoints DNS"
  title: "Cloud Endpoints DNS"
  version: "1.0.0"
paths: {}
host: "frontend.endpoints.${PROJECT}.cloud.goog"
x-google-endpoints:
- name: "frontend.endpoints.${PROJECT}.cloud.goog"
  target: "${GCLB_IP}"
EOF

gcloud --project=${PROJECT} endpoints services deploy dns-spec.yaml

```

### Configure Certificate Manager resources 

some notes 
- https://cloud.google.com/kubernetes-engine/docs/how-to/secure-gateway#restrictions_and_limitations
- https://cloud.google.com/kubernetes-engine/docs/how-to/secure-gateway#secure-using-certificate-manager

```
#create certificate 
gcloud --project=${PROJECT} certificate-manager certificates create edge2mesh-cert \
    --domains="frontend.endpoints.${PROJECT}.cloud.goog"

# create certificate map 
gcloud --project=${PROJECT} certificate-manager maps create edge2mesh-cert-map

# create certificate map entry
gcloud --project=${PROJECT} certificate-manager maps entries create edge2mesh-cert-map-entry \
    --map="edge2mesh-cert-map" \
    --certificates="edge2mesh-cert" \
    --hostname="frontend.endpoints.${PROJECT}.cloud.goog"
```

### Deploy Online Boutique
```
kubectl create namespace onlineboutique

kubectl label namespace onlineboutique istio-injection=enabled

curl -LO \
    https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/main/release/kubernetes-manifests.yaml

kubectl apply -f kubernetes-manifests.yaml -n onlineboutique


cat <<EOF > frontend-virtualservice.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: frontend-ingress
  namespace: onlineboutique
spec:
  hosts:
  - "frontend.endpoints.${PROJECT}.cloud.goog"
  gateways:
  - asm-ingress/asm-ingressgateway
  http:
  - route:
    - destination:
        host: frontend
        port:
          number: 80
EOF

kubectl apply -f frontend-virtualservice.yaml
```

### Create Gateway and HTTPRoutes

```
cat <<EOF > gateway.yaml
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1beta1
metadata:
  name: external-http
  namespace: asm-ingress
  annotations:
    networking.gke.io/certmap: edge2mesh-cert-map
spec:
  gatewayClassName: gke-l7-global-external-managed # gke-l7-gxlb
  listeners:
  - name: http # list the port only so we can redirect any incoming http requests to https
    protocol: HTTP
    port: 80
  - name: https
    protocol: HTTPS
    port: 443
  addresses:
  - type: NamedAddress
    value: ingress-ip
EOF

kubectl apply -f gateway.yaml

cat << EOF > default-httproute.yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: default-httproute
  namespace: asm-ingress
spec:
  parentRefs:
  - name: external-http
    namespace: asm-ingress
    sectionName: https
  rules:
  - matches:
    - path:
        value: /
    backendRefs:
    - name: asm-ingressgateway
      port: 443
EOF

kubectl apply -f default-httproute.yaml

# set up HTTP redirect as well
cat << EOF > default-httproute-redirect.yaml
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1beta1
metadata:
  name: http-to-https-redirect-httproute
  namespace: asm-ingress
spec:
  parentRefs:
  - name: external-http
    namespace: asm-ingress
    sectionName: http
  rules:
  - filters:
    - type: RequestRedirect
      requestRedirect:
        scheme: https
        statusCode: 301
EOF

kubectl apply -f default-httproute-redirect.yaml
```
**NOTE:** reconcilliation will take some time. `kubectl get gateway external-http -n asm-ingress -w` until `programmed=true`.
