# 🚲 GKE Poc Toolkit Demo: Whereami (Multi Cluster Networking)
 

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
cd gke-poc-config-sync
```

8. **Download a front end and back end set of release manifests into your ACM repo, then push to `main` branch.** This will effectively deploy the a frontend and backend WHEREAMI service to all of your GKE clusters.

```
cat <<EOF > default-namespace.yaml
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
kustomize build whereami/k8s-backend-overlay-example/ -o .
kustomize build whereami/k8s-frontend-clusterip-overlay-example/ -o .

rm -rf whereami
git add .
git commit -m "Deploy Whereami Front and backend services" 
git push origin main
```

