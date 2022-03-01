# 🚲 GKE Poc Toolkit Demo: Whereami (Multi Cluster Gateway and Anthos Service Mesh)
This demo shows you how to bootstrap three GKE clusters into a multi-cluster network designed for end to end encryption and a zero trust service mesh. GKE clusters, Anthos Service Mesh, and a multi-cluster gateway for ingress are installed with the gke-poc-toolkit followed by instuctions to build out a demo that shows how regional service failures are automatically recovered. For a more detailed step by step installation of these components please checkout this [repo](https://gitlab.com/asm7/secure-multicluster-ingress) where a good bit of this walk through borrows from.

## How to run 

1. **Go through the [GKE PoC Toolkit quickstart](https://github.com/GoogleCloudPlatform/gke-poc-toolkit#quickstart) up until the `gkekitctl create` and stop at step 6 (gkekitctl init).** 

2. **Copy `config.yaml` to wherever you're running the toolkit from.**

3. **Export vars and add them to your GKE POC toolkit config.yaml.**

``` bash 
export GKE_PROJECT_ID=<your-gke-clusters-project-id>
export VPC_PROJECT_ID=<your-sharedvpc-project-id>
sed -i '' -e "s/clustersProjectId: \"my-project\"/clustersProjectId: \"${GKE_PROJECT_ID}\"/g" config.yaml
sed -i '' -e "s/governanceProjectId: \"my-project\"/governanceProjectId: \"${GKE_PROJECT_ID}\"/g" config.yaml
sed -i '' -e "s/vpcProjectId: \"my-host-project\"/vpcProjectId: \"${VPC_PROJECT_ID}\"/g" config.yaml
```

4. **Run `./gkekitctl create --config config.yaml` from this directory.** This will take about 30 minutes to run.

5. **Connect to your newly-created GKE clusters.**

```bash
gcloud container clusters get-credentials gke-west --region us-west1 --project ${GKE_PROJECT_ID}
gcloud container clusters get-credentials gke-central --region us-central1 --project ${GKE_PROJECT_ID}
gcloud container clusters get-credentials gke-east --region us-east1 --project ${GKE_PROJECT_ID}
```

6. **We highly recommend installing [kubectx and kubens](https://github.com/ahmetb/kubectx) to switch kubectl contexts between clusters with ease. Once done, you can validate you clusters like so.**

```bash
kubectx ##Lists all clusters in your config
kubectx gke_${GKE_PROJECT_ID}_us-west1_gke-west
kubectl get nodes
kubectx gke_${GKE_PROJECT_ID}_us-east1_gke-east
kubectl get nodes
kubectx gke_${GKE_PROJECT_ID}_us-central1_gke-central
kubectl get nodes
```

*Expected output for each cluster*: 
```bash
NAME                                                  STATUS   ROLES    AGE   VERSION
gke-gke-central-linux-gke-toolkit-poo-12b0fa78-grhw   Ready    <none>   11m   v1.21.6-gke.1500
gke-gke-central-linux-gke-toolkit-poo-24d712a2-jm5g   Ready    <none>   11m   v1.21.6-gke.1500
gke-gke-central-linux-gke-toolkit-poo-6fb11d07-h6xb   Ready    <none>   11m   v1.21.6-gke.1500
```

7. **Clone your Anthos Config Management (ACM) sync repo.** 

```
gcloud source repos clone gke-poc-config-sync --project=$GKE_PROJECT_ID
export ACM_REPO_DIR=`pwd`
cd gke-poc-config-sync
export ACM_REPO_DIR=`pwd`
```

8. **Create static public IP and free DNS names in the cloud.goog domain using Cloud Endpoints DNS service for Bank of Anthos, Online Boutique and the whereami applications. [Learn more about configuring DNS on the cloud.goog domain](https://cloud.google.com/endpoints/docs/openapi/cloud-goog-dns-configure).**

```bash
gcloud compute addresses create gclb-ip --global
export GCLB_IP=`gcloud compute addresses describe gclb-ip --global --format="value(address)"`
echo -e "GCLB_IP is ${GCLB_IP}"

cat <<EOF > whereami-openapi.yaml
swagger: "2.0"
info:
  description: "Cloud Endpoints DNS"
  title: "Cloud Endpoints DNS"
  version: "1.0.0"
paths: {}
host: "whereami.endpoints.${GKE_PROJECT_ID}.cloud.goog"
x-google-endpoints:
- name: "whereami.endpoints.${GKE_PROJECT_ID}.cloud.goog"
  target: "${GCLB_IP}"
EOF

gcloud endpoints services deploy whereami-openapi.yaml
```

9. **Create a Google managed cert for the domain.**

``` bash
cat <<EOF > ${ACM_REPO_DIR}/managed-cert.yaml
apiVersion: networking.gke.io/v1beta2
kind: ManagedCertificate
metadata:
  name: whereami-managed-cert
  namespace: istio-system
  annotations:
    configsync.gke.io/cluster-name-selector: gke-east-membership
spec:
  domains:
  - "whereami.endpoints.${GKE_PROJECT_ID}.cloud.goog"
EOF
```

10. **Create Cloud Armor policies. [Google Cloud Armor](https://cloud.google.com/armor) provides DDoS defense and [customizable security policies](https://cloud.google.com/armor/docs/configure-security-policies) that you can attach to a load balancer through Ingress resources. In the following steps, you create a security policy that uses [preconfigured rules](https://cloud.google.com/armor/docs/rule-tuning#preconfigured_rules) to block cross-site scripting (XSS) attacks.

```bash
gcloud compute security-policies create gclb-fw-policy \
--description "Block XSS attacks"

gcloud compute security-policies rules create 1000 \
--security-policy gclb-fw-policy \
--expression "evaluatePreconfiguredExpr('xss-stable')" \
--action "deny-403" \
--description "XSS attack filtering"
```


10. **Configure certificate for GCLB to ASM ingress gateway**

```bash
openssl req -new -newkey rsa:4096 -days 365 -nodes -x509 \
-subj "/CN=frontend.endpoints.${GKE_PROJECT_ID}.cloud.goog/O=Edge2Mesh Inc" \
-keyout frontend.endpoints.${GKE_PROJECT_ID}.cloud.goog.key \
-out frontend.endpoints.${GKE_PROJECT_ID}.cloud.goog.crt

kubectl -n asm-gateways create secret tls edge2mesh-credential \
--key=frontend.endpoints.${GKE_PROJECT_ID}.cloud.goog.key \
--cert=frontend.endpoints.${GKE_PROJECT_ID}.cloud.goog.crt --dry-run=client -o yaml >> ${ACM_REPO_DIR}/backendcert.yaml
```

11. **Download a front end and back end set of release manifests into your ACM repo, then push to `main` branch. This will effectively deploy the a frontend and backend WHEREAMI service to all of your GKE clusters.**

```bash
cat <<EOF > ${ACM_REPO_DIR}/default-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: default
  labels:
    istio.io/rev: asm-managed
EOF
git add .
git commit -m "Update default namespace with istio label" 
git push origin main

git clone https://github.com/knee-berts/gke-whereami.git whereami
kustomize build whereami/k8s-backend-overlay-example/ -o ${ACM_REPO_DIR}
kustomize build whereami/k8s-frontend-clusterip-overlay-example/ -o ${ACM_REPO_DIR}

rm -rf whereami
git add .
git commit -m "Deploy Whereami Front and backend services" 
git push origin main
```

12. **Copy ASM Ingress Gateway, MCI, MCS, and cert configs to ACM repo and push them tp `main` branch.**

```bash
cat <<EOF > ${ACM_REPO_DIR}/asm-ingressgateway-external.yaml
apiVersion: v1
kind: Service
metadata:
  name: asm-ingressgateway-xlb
  namespace: asm-gateways
spec:
  type: ClusterIP
  selector:
    asm: ingressgateway-xlb
  ports:
  - port: 80
    name: http
  - port: 443
    name: https
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: asm-ingressgateway-xlb
  namespace: asm-gateways
spec:
  selector:
    matchLabels:
      asm: ingressgateway-xlb
  template:
    metadata:
      annotations:
        # This is required to tell Anthos Service Mesh to inject the gateway with the
        # required configuration.
        inject.istio.io/templates: gateway
      labels:
        asm: ingressgateway-xlb
        # asm.io/rev: ${ASM_LABEL} # This is required only if the namespace is not labeled.
    spec:
      containers:
      - name: istio-proxy
        image: auto # The image will automatically update each time the pod starts.
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: asm-ingressgateway-sds
  namespace: asm-gateways
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: asm-ingressgateway-sds
  namespace: asm-gateways
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: asm-ingressgateway-sds
subjects:
  - kind: ServiceAccount
    name: default
EOF

export WHEREAMI_MANAGED_CERT=$(kubectl --context gke_${GKE_PROJECT_ID}_us-east1_gke-east -n istio-system get managedcertificate whereami-managed-cert -ojsonpath='{.status.certificateName}')

cat <<EOF > ${ACM_REPO_DIR}/mci.yaml
apiVersion: networking.gke.io/v1beta1
kind: MultiClusterIngress
metadata:
  name: asm-ingressgateway-xlb-multicluster-ingress
  namespace: asm-gateways
  annotations:
    networking.gke.io/static-ip: "${GCLB_IP}"
    networking.gke.io/pre-shared-certs: "${WHEREAMI_MANAGED_CERT}"
spec:
  template:
    spec:
      backend:
        serviceName: asm-ingressgateway-xlb-multicluster-svc
        servicePort: 443
EOF

cat <<EOF > ${ACM_REPO_DIR}/mcs.yaml
apiVersion: networking.gke.io/v1beta1
kind: MultiClusterService
metadata:
  name: asm-ingressgateway-xlb-multicluster-svc
  namespace: asm-gateways
  annotations:
    beta.cloud.google.com/backend-config: '{"ports": {"443":"asm-ingress-xlb-config"}}'
    networking.gke.io/app-protocols: '{"http2":"HTTP2"}'
spec:
  template:
    spec:
      selector:
        asm: ingressgateway-xlb
      ports:
      - name: http2
        protocol: TCP
        port: 443 # Port the Service listens on
EOF

cat <<EOF > ${ACM_REPO_DIR}/backend-config.yaml
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: asm-ingress-xlb-config
  namespace: asm-gateways
spec:
  healthCheck:
    type: HTTP
    port: 15021
    requestPath: /healthz/ready
  securityPolicy:
    name: "gclb-fw-policy"
EOF

git add .
git commit -m "Deploy multi cluster components." 
git push origin main
```

12. **This while loop will run until the managed cert is provisioned, it can take up to 30 mins. In the meantime, let's go over how we have achieved end to end encryption. [source](https://gitlab.com/asm7/secure-multicluster-ingress#end-to-end-multicluster-ingress-encryption)**

```bash
## Check the managed cert provisions status, we want it to return "ACTIVE"
while [ `gcloud beta compute ssl-certificates describe ${WHEREAMI_MANAGED_CERT} --format='value(managed.status)'` != "ACTIVE" ]; do echo "Cert is not Active" sleep 10; done
```

        There are three legs of this ingress:
        
        1. The first leg is from the client to the Google load balancer (GCLB). This leg uses the Google managed certificate or any certificate that is trusted by external clients.
        2. The second leg is from the Google load balancer to the ASM ingress gateway. You can use any certificate between GCLB and ASM ingress gateway. In this tutorial you create a self-signed certificate. In production environments, you can use any PKI for this certificate.
        3. The third leg is from the ASM ingress gateway to the desired Service. Traffic between ASM ingress gateways and all mesh services can be encrypted using mTLS. Mesh CA is the certificate authority that performs worload certificate management.

13. **Validate Gateway regional failover

14. **Validate Service to Service regional failover




