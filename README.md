```
# mesh-cross-domain
#setup E2M on first project

export PROJECT=chilm-mesh-xdom-a
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT} --format="value(projectNumber)")
gcloud config set project ${PROJECT}
mkdir -p ${HOME}/edge-to-mesh
cd ${HOME}/edge-to-mesh
export WORKDIR=`pwd`
touch edge2mesh_kubeconfig
export KUBECONFIG=${WORKDIR}/edge2mesh_kubeconfig
export CLUSTER_NAME=source-mesh
export CLUSTER_1_NAME=source-mesh
export CLUSTER_LOCATION=us-east4
gcloud services enable container.googleapis.com
gcloud container --project ${PROJECT} clusters create-auto \
   ${CLUSTER_NAME} --region ${CLUSTER_LOCATION} --release-channel rapid
gcloud services enable mesh.googleapis.com
gcloud container fleet mesh enable
gcloud container fleet memberships register ${CLUSTER_NAME} \
  --gke-cluster ${CLUSTER_LOCATION}/${CLUSTER_NAME}
gcloud container clusters update ${CLUSTER_NAME} --project ${PROJECT} --region ${CLUSTER_LOCATION} --update-labels mesh_id=proj-${PROJECT_NUMBER}
gcloud container fleet mesh update \
  --management automatic \
  --memberships ${CLUSTER_NAME}
kubectl create namespace asm-ingress-ext
kubectl label namespace asm-ingress-ext istio-injection=enabled

openssl req -new -newkey rsa:4096 -days 365 -nodes -x509 \
 -subj "/CN=frontend.endpoints.${PROJECT}.cloud.goog/O=Edge2Mesh Inc" \
 -keyout frontend.endpoints.${PROJECT}.cloud.goog.key \
 -out frontend.endpoints.${PROJECT}.cloud.goog.crt

kubectl -n asm-ingress-ext create secret tls edge2mesh-credential \
 --key=frontend.endpoints.${PROJECT}.cloud.goog.key \
 --cert=frontend.endpoints.${PROJECT}.cloud.goog.crt

mkdir -p ${WORKDIR}/asm-ig/base
cat <<EOF > ${WORKDIR}/asm-ig/base/kustomization.yaml
resources:
  - github.com/GoogleCloudPlatform/anthos-service-mesh-samples/docs/ingress-gateway-asm-manifests/base
EOF

mkdir ${WORKDIR}/asm-ig/variant
cat <<EOF > ${WORKDIR}/asm-ig/variant/role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: asm-ingressgateway
  namespace: asm-ingress-ext
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
EOF

cat <<EOF > ${WORKDIR}/asm-ig/variant/rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: asm-ingressgateway
  namespace: asm-ingress-ext
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: asm-ingressgateway
subjects:
  - kind: ServiceAccount
    name: asm-ingressgateway
EOF

cat <<EOF > ${WORKDIR}/asm-ig/variant/service-proto-type.yaml
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

cat <<EOF > ${WORKDIR}/asm-ig/variant/gateway.yaml
apiVersion: networking.istio.io/v1beta1
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

cat <<EOF > ${WORKDIR}/asm-ig/variant/kustomization.yaml
namespace: asm-ingress-ext
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

kubectl apply -k ${WORKDIR}/asm-ig/variant

cat <<EOF >${WORKDIR}/ingress-gateway-healthcheck.yaml
apiVersion: networking.gke.io/v1
kind: HealthCheckPolicy
metadata:
  name: ingress-gateway-healthcheck
  namespace: asm-ingress-ext
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

kubectl apply -f ${WORKDIR}/ingress-gateway-healthcheck.yaml

gcloud compute addresses create e2m-gclb-ip --global
export GCLB_IP=$(gcloud compute addresses describe e2m-gclb-ip \
  --global --format "value(address)")
echo ${GCLB_IP}

cat <<EOF > ${WORKDIR}/dns-spec.yaml
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

gcloud endpoints services deploy ${WORKDIR}/dns-spec.yaml
gcloud services enable certificatemanager.googleapis.com --project=${PROJECT}
gcloud --project=${PROJECT} certificate-manager certificates create edge2mesh-cert \
    --domains="frontend.endpoints.${PROJECT}.cloud.goog"
gcloud --project=${PROJECT} certificate-manager maps create edge2mesh-cert-map
gcloud --project=${PROJECT} certificate-manager maps entries create edge2mesh-cert-map-entry \
    --map="edge2mesh-cert-map" \
    --certificates="edge2mesh-cert" \
    --hostname="frontend.endpoints.${PROJECT}.cloud.goog"

cat <<EOF > ${WORKDIR}/gke-gateway.yaml
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1beta1
metadata:
  name: external-http
  namespace: asm-ingress-ext
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
    value: e2m-gclb-ip # reference the static IP created earlier
EOF

kubectl apply -f ${WORKDIR}/gke-gateway.yaml

cat << EOF > ${WORKDIR}/default-httproute.yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: default-httproute
  namespace: asm-ingress-ext
spec:
  parentRefs:
  - name: external-http
    namespace: asm-ingress-ext
    sectionName: https
  rules:
  - matches:
    - path:
        value: /
    backendRefs:
    - name: asm-ingressgateway
      port: 443
EOF

kubectl apply -f ${WORKDIR}/default-httproute.yaml

cat << EOF > ${WORKDIR}/default-httproute-redirect.yaml
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1beta1
metadata:
  name: http-to-https-redirect-httproute
  namespace: asm-ingress-ext
spec:
  parentRefs:
  - name: external-http
    namespace: asm-ingress-ext
    sectionName: http
  rules:
  - filters:
    - type: RequestRedirect
      requestRedirect:
        scheme: https
        statusCode: 301
EOF

kubectl apply -f ${WORKDIR}/default-httproute-redirect.yaml

## second cluster (target for xdom service)

export PROJECT=chilm-mesh-xdom-b
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT} --format="value(projectNumber)")
gcloud config set project ${PROJECT}

mkdir -p ${HOME}/edge-to-mesh-target
cd ${HOME}/edge-to-mesh-target
export WORKDIR=`pwd`

export CLUSTER_2_NAME=target-mesh
export CLUSTER_LOCATION=us-central1
gcloud services enable container.googleapis.com
gcloud container --project ${PROJECT} clusters create-auto \
   ${CLUSTER_2_NAME} --region ${CLUSTER_LOCATION} --release-channel rapid
gcloud services enable mesh.googleapis.com
gcloud container fleet mesh enable
gcloud container fleet memberships register ${CLUSTER_2_NAME} \
  --gke-cluster ${CLUSTER_LOCATION}/${CLUSTER_2_NAME}
gcloud container clusters update ${CLUSTER_2_NAME} --project ${PROJECT} --region ${CLUSTER_LOCATION} --update-labels mesh_id=proj-${PROJECT_NUMBER}
gcloud container fleet mesh update \
  --management automatic \
  --memberships ${CLUSTER_2_NAME}
kubectl create namespace asm-ingress-int
kubectl label namespace asm-ingress-int istio-injection=enabled

gcloud compute addresses create ing-gclb-ip --region {CLUSTER_LOCATION}
export GCNLB_IP=$(gcloud compute addresses describe ing-gclb-ip \
  --global --format "value(address)")
echo ${GCNLB_IP}

cat <<EOF > ${WORKDIR}/dns-spec.yaml
swagger: "2.0"
info:
  description: "Cloud Endpoints DNS"
  title: "Cloud Endpoints DNS"
  version: "1.0.0"
paths: {}
host: "ig.endpoints.${PROJECT}.cloud.goog"
x-google-endpoints:
- name: "ig.endpoints.${PROJECT}.cloud.goog"
  target: "${GCNLB_IP}"
EOF

gcloud endpoints services deploy ${WORKDIR}/dns-spec.yaml

mkdir -p ${WORKDIR}/asm-ig/base
cat <<EOF > ${WORKDIR}/asm-ig/base/kustomization.yaml
resources:
  - github.com/GoogleCloudPlatform/anthos-service-mesh-samples/docs/ingress-gateway-asm-manifests/base
EOF


mkdir ${WORKDIR}/asm-ig/variant

cat <<EOF > ${WORKDIR}/asm-ig/variant/service-proto-type.yaml
apiVersion: v1
kind: Service
metadata:
  name: asm-ingressgateway
spec:
  type: LoadBalancer
  loadBalancerIP: ${GCNLB_IP}
EOF

cat <<EOF > ${WORKDIR}/asm-ig/variant/gateway.yaml
apiVersion: networking.istio.io/v1beta1
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
      mode: ISTIO_MUTUAL
EOF

cat <<EOF > ${WORKDIR}/asm-ig/variant/kustomization.yaml
namespace: asm-ingress-int
resources:
- ../base
patches:
- path: service-proto-type.yaml
  target:
    kind: Service
- path: gateway.yaml
  target:
    kind: Gateway
EOF

kubectl --context=${CLUSTER_2_NAME} apply -k ${WORKDIR}/asm-ig/variant

# Helm is the state-of-the-art when deploying Istio/ASM gateways
# helm repo add istio https://istio-release.storage.googleapis.com/charts
# helm repo update

# Deploy a gateway to the target cluster
# Make sure it creates a Service of type LoadBalancer
# * It's an NLB
# * It uses the IP we reserved
# helm install "asm-ingressgateway" istio/gateway -n "asm-ingress-int" \
#    --set revision="asm-managed-rapid" \
#    --set service.loadBalancerIP=${GCNLB_IP}

kubectl --context=${CLUSTER_1_NAME} create ns frontend
kubectl --context=${CLUSTER_1_NAME} label namespace frontend istio-injection=enabled
kubectl --context=${CLUSTER_2_NAME} create ns backend
kubectl --context=${CLUSTER_2_NAME} label namespace backend istio-injection=enabled
mkdir -p ${WORKDIR}/whereami-backend/base

cat <<EOF > ${WORKDIR}/whereami-backend/base/kustomization.yaml 
resources:
  - github.com/GoogleCloudPlatform/kubernetes-engine-samples/quickstarts/whereami/k8s
EOF

mkdir ${WORKDIR}/whereami-backend/variant

cat <<EOF > ${WORKDIR}/whereami-backend/variant/cm-flag.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: whereami
data:
  BACKEND_ENABLED: "False" # assuming you don't want a chain of backend calls
  METADATA:        "backend"
EOF

cat <<EOF > ${WORKDIR}/whereami-backend/variant/service-type.yaml 
apiVersion: "v1"
kind: "Service"
metadata:
  name: "whereami"
spec:
  type: ClusterIP
EOF

cat <<EOF > ${WORKDIR}/whereami-backend/variant/kustomization.yaml 
nameSuffix: "-backend"
namespace: backend
commonLabels:
  app: whereami-backend
resources:
- ../base
patches:
- path: cm-flag.yaml
  target:
    kind: ConfigMap
- path: service-type.yaml
  target:
    kind: Service
EOF

kubectl --context=${CLUSTER_2_NAME} apply -k ${WORKDIR}/whereami-backend/variant

mkdir -p ${WORKDIR}/whereami-frontend/base

cat <<EOF > ${WORKDIR}/whereami-frontend/base/kustomization.yaml 
resources:
  - github.com/GoogleCloudPlatform/kubernetes-engine-samples/quickstarts/whereami/k8s
EOF

mkdir whereami-frontend/variant

cat <<EOF > ${WORKDIR}/whereami-frontend/variant/cm-flag.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: whereami
data:
  BACKEND_ENABLED: "True"
  BACKEND_SERVICE:        "http://ig.endpoints.${PROJECT}.cloud.goog/"
EOF

cat <<EOF > ${WORKDIR}/whereami-frontend/variant/service-type.yaml 
apiVersion: "v1"
kind: "Service"
metadata:
  name: "whereami"
spec:
  type: ClusterIP
EOF

cat <<EOF > ${WORKDIR}/whereami-frontend/variant/kustomization.yaml 
nameSuffix: "-frontend"
namespace: frontend
commonLabels:
  app: whereami-frontend
resources:
- ../base
patches:
- path: cm-flag.yaml
  target:
    kind: ConfigMap
- path: service-type.yaml
  target:
    kind: Service
EOF

kubectl --context=${CLUSTER_1_NAME} apply -k ${WORKDIR}/whereami-frontend/variant

cat << EOF > ${WORKDIR}/frontend-vs.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: whereami-vs
  namespace: frontend
spec:
  gateways:
  - asm-ingress-ext/asm-ingressgateway
  hosts:
  - 'frontend.endpoints.${SOURCE_PROJECT}.cloud.goog'
  http:
  - route:
    - destination:
        host: whereami-frontend
        port:
          number: 80
EOF

kubectl --context=${CLUSTER_1_NAME} apply -f ${WORKDIR}/frontend-vs.yaml

<!-- kubectl --context=${CLUSTER_2_NAME} apply -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      name: asm-ingressgateway
      namespace: asm-ingress-int
    spec:
      selector:
        istio: asm-ingressgateway
      servers:
      - port:
          number: 443
          name: https
          protocol: HTTPS
        hosts:
        - "*"
        tls:
          mode: ISTIO_MUTUAL
EOF -->

kubectl apply --context=${CLUSTER_2_NAME} -f - <<EOF
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: whereami-vs
      namespace: backend
    spec:
      hosts:
      - "*"
      gateways:
      - asm-ingress-int/asm-ingressgateway
      http:
      - match:
        - uri:
            exact: /
        route:
        - destination:
            host: whereami-backend
            port:
              number: 80
        headers:
          request:
            set:
              san-uri-seen-at-gateway: "%DOWNSTREAM_PEER_URI_SAN%"
EOF

kubectl apply --context=${CLUSTER_1_NAME} -f - <<EOF
    apiVersion: networking.istio.io/v1beta1
    kind: ServiceEntry
    metadata:
      name: backend-remote
      namespace: frontend
    spec:
      #addresses:
      #  - 240.240.1.2 # Fake address
      #endpoints:
      #  - address: ${ILB_IP_1}
      #  - address: ${ILB_IP_2}
      hosts:
        - ig.endpoints.chilm-mesh-xdom-b.cloud.goog
      location: MESH_EXTERNAL
      ports:
        - name: http-80
          number: 80
          protocol: HTTP
          targetPort: 443
        - name: https-443
          number: 443
          protocol: HTTPS
      resolution: DNS
EOF

kubectl apply --context=${CLUSTER_1_NAME} -f - <<EOF
    apiVersion: networking.istio.io/v1beta1
    kind: DestinationRule
    metadata:
      name: backend-remote
      namespace: frontend
    spec:
      host: ig.endpoints.chilm-mesh-xdom-b.cloud.goog
      trafficPolicy:
        tls:
          mode: ISTIO_MUTUAL
EOF

kubectl apply --context=${CLUSTER_2_NAME} -f - <<EOF
    apiVersion: security.istio.io/v1beta1
    kind: AuthorizationPolicy
    metadata:
      name: backend-ap
      namespace: backend
    spec:
      action: ALLOW
      rules:
      - from:
        - source:
            namespaces: ["asm-ingress-int"]
        when:
        - key: request.headers[San-Uri-Seen-At-Gateway]
          values: ["spiffe://chilm-mesh-xdom-a.svc.id.goog/ns/frontend/sa/whereami-frontend"]
EOF

kubectl apply --context=${CLUSTER_1_NAME} --namespace istio-system -f - <<EOF
    apiVersion: "security.istio.io/v1beta1"
    kind: "PeerAuthentication"
    metadata:
      name: "default"
    spec:
      mtls:
        mode: STRICT
EOF

kubectl apply --context=${CLUSTER_2_NAME} --namespace istio-system -f - <<EOF
    apiVersion: "security.istio.io/v1beta1"
    kind: "PeerAuthentication"
    metadata:
      name: "default"
    spec:
      mtls:
        mode: STRICT
EOF
```




