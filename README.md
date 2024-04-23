
# mesh-cross-domain
### setup Edge to Mesh on first project

Expose Mesh Ingress Gateway as external L7 managed Gateway
```
export PROJECT_1=chilm-mesh-xdom-a
export PROJECT_1_NUMBER=$(gcloud projects describe ${PROJECT_1} --format="value(projectNumber)")
gcloud config set project ${PROJECT_1}
mkdir -p ${HOME}/edge-to-mesh-a
cd ${HOME}/edge-to-mesh-a
export WORKDIR=`pwd`

touch edge2mesh_kubeconfig
export KUBECONFIG=${WORKDIR}/edge2mesh_kubeconfig

export CLUSTER_1_NAME=source-mesh
export CLUSTER_1_LOCATION=us-east4
gcloud services enable container.googleapis.com
```
Create a GKE Autopilot cluster:

**Note:** Assumes a VPC named `default` exists
```
gcloud container --project ${PROJECT_1} clusters create-auto \
   ${CLUSTER_1_NAME} --region ${CLUSTER_1_LOCATION} --release-channel rapid

export CONTEXT_1=$(kubectl config current-context) 

gcloud services enable mesh.googleapis.com
gcloud container fleet mesh enable
gcloud container fleet memberships register ${CLUSTER_1_NAME} \
  --gke-cluster ${CLUSTER_1_LOCATION}/${CLUSTER_1_NAME}
gcloud container clusters update ${CLUSTER_1_NAME} --project ${PROJECT_1} --region ${CLUSTER_1_LOCATION} --update-labels mesh_id=proj-${PROJECT_1_NUMBER}
gcloud container fleet mesh update \
  --management automatic \
  --memberships ${CLUSTER_1_NAME}
kubectl create namespace asm-ingress-ext
kubectl label namespace asm-ingress-ext istio-injection=enabled
```
We'll create a self-signed cert to encrypt traffic between the LB and the backends
```
openssl req -new -newkey rsa:4096 -days 365 -nodes -x509 \
 -subj "/CN=frontend.endpoints.${PROJECT}.cloud.goog/O=Edge2Mesh Inc" \
 -keyout frontend.endpoints.${PROJECT}.cloud.goog.key \
 -out frontend.endpoints.${PROJECT}.cloud.goog.crt

kubectl -n asm-ingress-ext create secret tls edge2mesh-credential \
 --key=frontend.endpoints.${PROJECT}.cloud.goog.key \
 --cert=frontend.endpoints.${PROJECT}.cloud.goog.crt
```
Now we ready a deployment of Mesh Ingress Gateway
**Note:** check mesh is provisioned via `gcloud container fleet mesh describe`
```
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

```
apply it to our cluster
```
kubectl --context=${CONTEXT_1} apply -k ${WORKDIR}/asm-ig/variant

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

kubectl --context=${CONTEXT_1} apply -f ${WORKDIR}/ingress-gateway-healthcheck.yaml

gcloud compute addresses create e2m-gclb-ip --global
export GCLB_IP=$(gcloud compute addresses describe e2m-gclb-ip \
  --global --format "value(address)")
echo ${GCLB_IP}

#This creates a DNS entry
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

kubectl --context=${CONTEXT_1} apply -f ${WORKDIR}/gke-gateway.yaml

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

kubectl --context=${CONTEXT_1} apply -f ${WORKDIR}/default-httproute.yaml

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

kubectl --context=${CONTEXT_1} apply -f ${WORKDIR}/default-httproute-redirect.yaml
```
### Create second cluster (target for remote service call)

We put this in a different project with no connectivity to the first

```
export PROJECT_2=chilm-mesh-xdom-b
export PROJECT_2_NUMBER=$(gcloud projects describe ${PROJECT_2} --format="value(projectNumber)")
gcloud config set project ${PROJECT_2}

mkdir -p ${HOME}/edge-to-mesh-target
cd ${HOME}/edge-to-mesh-target
export WORKDIR=`pwd`

export CLUSTER_2_NAME=target-mesh
export CLUSTER_2_LOCATION=us-central1
gcloud services enable container.googleapis.com
gcloud container --project ${PROJECT_2} clusters create-auto \
   ${CLUSTER_2_NAME} --region ${CLUSTER__2_LOCATION} --release-channel rapid

export CONTEXT_2=$(kubectl config current-context)

gcloud services enable mesh.googleapis.com
gcloud container fleet mesh enable
gcloud container fleet memberships register ${CLUSTER_2_NAME} \
  --gke-cluster ${CLUSTER_2_LOCATION}/${CLUSTER_2_NAME}
gcloud container clusters update ${CLUSTER_2_NAME} --project ${PROJECT_2} --region ${CLUSTER_2_LOCATION} --update-labels mesh_id=proj-${PROJECT_2_NUMBER}
gcloud container fleet mesh update \
  --management automatic \
  --memberships ${CLUSTER_2_NAME}
kubectl create namespace asm-ingress-int
kubectl label namespace asm-ingress-int istio-injection=enabled

```
For the sake of similicity we'll expose the "internal" mesh Ingress Gateway exernally via a L4 NLB
L7 can't be used since we want to enforce mTLS between meshes, so mTLS from source service will
be terminated at the target mesh ingress gateway
```
gcloud compute addresses create ing-gclb-ip --region {CLUSTER_LOCATION}
export GCNLB_IP=$(gcloud compute addresses describe ing-gclb-ip \
  --global --format "value(address)")
echo ${GCNLB_IP}
```
We create a DNS entry, but if L4 ILB is created and exposed via PSC an internal zone Cloud DNS entry would
be created for the "consumer" PSC IP address
```
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
```
Now let's ready the target mesh ingress gateway deployment.
Note - the Kustomize for the service object can be updated to create an "Internal" L4 NLB
and add annotation for "global access" as needed to meet requirements for exposing
a PSC endpoint
```
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
  ports:
  - name: status-port
    port: 15021
    protocol: TCP
    targetPort: 15021
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  - name: https
    port: 443
    targetPort: 8443
    protocol: TCP
  type: LoadBalancer
  loadBalancerIP: ${GCNLB_IP}

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

kubectl --context=${CONTEXT_2} apply -k ${WORKDIR}/asm-ig/variant
```
Now we'll deploy a `wherami` frontend in the source cluster that willl call
`whereami` backend in the target cluster
```
kubectl --context=${CONTEXT_1} create ns frontend
kubectl --context=${CONTEXT_1} label namespace frontend istio-injection=enabled
kubectl --context=${CONTEXT_2} create ns backend
kubectl --context=${CONTEXT_2} label namespace backend istio-injection=enabled

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

kubectl --context=${CONTEXT_2} apply -k ${WORKDIR}/whereami-backend/variant

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
  BACKEND_SERVICE: "http://ig.endpoints.${PROJECT_2}.cloud.goog/"
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

kubectl --context=${CONTEXT_1} apply -k ${WORKDIR}/whereami-frontend/variant

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
  - 'frontend.endpoints.${PROJECT_1}.cloud.goog'
  http:
  - route:
    - destination:
        host: whereami-frontend
        port:
          number: 80
EOF

kubectl --context=${CONTEXT_1} apply -f ${WORKDIR}/frontend-vs.yaml
```
**Note** we could expose multiple services by creating VS per target service.

This one just points to "/" path

**Note** There are 2 mTLS "hops". On the first hop we grab the identity and put it into
a new header (always replacing any existing value) so we may use it on an `AuthorizationPolicy`
in the second mTLS "hop" (from target IG to target service).
```
kubectl apply --context=${CONTEXT_2} -f - <<EOF
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

#ServiceEntry pointing to the DNS entry for the target IG
kubectl apply --context=${CONTEXT_1} -f - <<EOF
    apiVersion: networking.istio.io/v1beta1
    kind: ServiceEntry
    metadata:
      name: backend-remote
      namespace: frontend
    spec:
      hosts:
        - ig.endpoints.${PROJECT_2}.cloud.goog
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

#Enforce mTLS for calls to the ServiceEntry
kubectl apply --context=${CONTEXT_1} -f - <<EOF
    apiVersion: networking.istio.io/v1beta1
    kind: DestinationRule
    metadata:
      name: backend-remote
      namespace: frontend
    spec:
      host: ig.endpoints.${PROJECT_2}.cloud.goog
      trafficPolicy:
        tls:
          mode: ISTIO_MUTUAL
EOF

#Note: this policy references the header we set on the target IG. Typically this would be
combined with a **default** ALLOW none AuthPol for the target cluster

kubectl apply --context=${CONTEXT_2} -f - <<EOF
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

kubectl apply --context=${CONTEXT_1} --namespace istio-system -f - <<EOF
    apiVersion: "security.istio.io/v1beta1"
    kind: "PeerAuthentication"
    metadata:
      name: "default"
    spec:
      mtls:
        mode: STRICT
EOF

kubectl apply --context=${CONTEXT_2} --namespace istio-system -f - <<EOF
    apiVersion: "security.istio.io/v1beta1"
    kind: "PeerAuthentication"
    metadata:
      name: "default"
    spec:
      mtls:
        mode: STRICT
EOF
```
Now we can curl the frontend service as observe a successful call to the backend

`curl -s https://frontend.endpoints.${PROJECT_1}.cloud.goog`
