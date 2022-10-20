# Greenhouse Sandbox Cluster Readme
This is an F5 Greenhouse variant of the Sandbox cluster. It adds tooling for Gitlab based deployments and Auto-Devops,
and eventually e.g. the Threatstack agent or other specific Greenhouse requirements.

**Quick Links**
* [Sandbox Cluster Documentation](https://clhain.github.io/sandbox/) - Detailed usage guide for the F5 Sandbox.
* [Sandbox Cluster Repo](https://github.com/clhain/sandbox/) - Source code contianing F5 Sandbox Apps and Bundle definitions.

**Repo Layout**
* **/cluster/** - Contains pre-configured porter files for cluster installation based on your selection in the Applab UI.
* **/.porter/** - Contains porter config template for use during install.
* **/cluster/params.yaml** - Most of the configuration settings for the porter bundle, update values here if needed.
* **/gitlab-ci/sandbox_actions.yml** - The Gitlab Runner Pipeline which can be used to deploy the Cluster via Porter.

## Deploy The Cluster
The cluster can be deployed either using the Gitlab Runner, or via the porter CLI. If you use the runner, you should
supply mongodb connection information in order for your state files to be maintained between jobs. If you use the
local porter cli, you can skip the mongo creds but you won't be able to collaborate with other users.

These instructions are for a modified version of the [All In One (GKE) Sandbox Deployment](https://clhain.github.io/sandbox/installation/all-in-one-gke/).
More detailed instructions can be found at that link, with Applab specific modifications here.

### Create A Deployer Service Account Key
Example account creation and permission commands (must be run by a user with permissions to create service accounts):

```text
gcloud iam service-accounts create deployer-sa \
    --description="Service Account Used with Porter to deploy GCP assets" \
    --display-name="Porter Deploy Service Account"

gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:deployer-sa@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/iam.serviceAccountUser"

gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:deployer-sa@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/container.admin"

gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:deployer-sa@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/container.clusterAdmin"

gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:deployer-sa@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/compute.networkAdmin"

gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:deployer-sa@PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/compute.admin"
```

If you didn't already specify a pre-existing Service Account for the GKE nodes, you can create one like this,
then update the parameter `cluster_service_account_name` in `cluster/params.yaml`:

```text
gcloud iam service-accounts create sandbox-cluster-node-sa \
    --description="Service Account For GKE Cluster Nodes" \
    --display-name="Cluster Node Service Account"
```

### Using GitLab Runner
The deployment pipeline can automatically deploy and update the cluster once the following variables are set:

1. Add a mongodb connection string variable for porter to maintain state between runs:
   * GitLab: Settings -> CI/CD -> Variables -> Add key **PORTER_MONGO_CONNECTION_STRING** and value (your string e.g. mongodb+srv://USER_HERE:PASSWORD_HERE@YOUR_HOST/?retryWrites=true&w=majority)
2. Add the BASE64 encoded copy of your deployer service account key:
   * GitLab: Settings -> CI/CD -> Variables -> Add key **BASE64_GOOGLE_CREDENTIALS** and value (cat YOUR_GLOUD_KEY | base64 -w0)
3. Add the OIDC Client Secret if you're using an external IDP:
   * GitLab: Settings -> CI/CD -> Variables -> Add key **OIDC_CLIENT_SECRET** and value. Leve this out for local auth via dex.

Manually trigger the pipeline, and [set the gitlab variable](https://docs.gitlab.com/ee/ci/variables/#override-a-variable-when-running-a-pipeline-manually)
**"UPDATE_CLUSTER"** to **"true"** from CI/CD -> Pipelines -> Run Pipeline.

### Using Porter CLI
Optionally, if you want to connect your local instance to a shared mongodb database, run:

```text
export PORTER_MONGO_CONNECTION_STRING=mongodb+srv://USER_HERE:PASSWORD_HERE@YOUR_HOST/?retryWrites=true&w=majority
cp ./.porter/config.toml ~/.porter/config.toml
```

* Install porter
* Set the **OIDC_CLIENT_SECRET** variable to your value (or empty string) `export OIDC_CLIENT_SECRET=your value`
* place the **GCP Deployer service account key** in /tmp/gcloud.key (or update values in cluster/params.yaml)
* run porter parameters apply cluster/params.yaml
* run porter credentials apply cluster/creds.yaml
* run porter installation apply cluster/installation.yaml --force

## Verify the Cluster Installation
After the pipeline completes, [Connect to the cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl)
using the gcloud connection string from the gcloud portal.

The actual application install can take an additional 10-20 minutes, and you can follow along via the 
[Verification](https://clhain.github.io/sandbox/installation/quick-start/#verification) section of the sandbox
documentation to ensure the deployment rolls out. Note: You'll need to add the Gitlab Agent token (see the next section) before
the gitlab-agent app will enter the Healthy state.

Once everything is up and running you can browse to:

* grafana.your_cluster_domain
* argocd.your_cluster_domain

Login credentials for Dex local auth (admin@example.com) can be found with:

```text
kubectl get secret -n oauth-proxy oauth-proxy-creds -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

## Authenticate The Cluster With GitLab
In Gitlab, go to Infrastructure -> Kubernetes Cluster -> Connect a cluster, then select or create an agent token.

### Regular Secrets With Kubectl
You can create the requisite secret and (optionally namespace if the ArgoCD Installation is still running) via the command line:

```text
kubectl create namespace gitlab-agent
kubectl create -n gitlab-agent secret generic gitlab-agent-token --from-literal=token=YOUR_GITLAB_AGENT_TOKEN
```

### With Sealed Secrets
Alternatively, you can use the cluster's instance of [sealed-secrets](https://github.com/bitnami-labs/sealed-secrets) to generate
an encrypted version of the secret that will only work with this cluster and is safe to commit to your code repo:

Create a file e.g. /tmp/my_raw_secret.yaml, and add your token:
```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: gitlab-agent-token
  namespace: gitlab-agent
stringData:
  token: YOUR_TOKEN
```

Then use the cluster sealed secrets to encrypt:
```text
kubectl apply -f /tmp/my_raw_secret.yaml --dry-run -o json | kubeseal --controller-name sealed-secrets --controller-namespace sealed-secrets | yq -P '.' > my-sealed-secret.yaml
```

The sealed secret can then be applied to the cluster using `kubectl create namespace gitlab-agent && kubectl apply -f my-sealed-secret.yaml`.

## Deploy Code To The Cluster

### With AutoDevops
Add the autodevops template to your `.gitlab-ci.yml` file under the `include` key, e.g.:

```yaml
include:
  - template: Auto-DevOps.gitlab-ci.yml      # Add this line
  - local: /.gitlab-ci/sandbox_actions.yml
    rules:
      - if: $UPDATE_CLUSTER == "true"
```

Add the following variables to your Gitlab Pipeline Config:

#TODO: Make this suck less

KUBE_CONTEXT = your gitlab org and repo name + the name of your gitlab agent... e.g. f5/greenhouse/my-cool-repo:my-cool-gitlab-agent
KUBE_INGRESS_BASE_DOMAIN = same as cluster domain... You'll also need a DNS record for your-repo-name.your_cluseter_domain.
KUBE_NAMESPACE = whatever namespace name you want to deploy into.

Add some code.

### Use With ArgoCd
TODO


### Uninstall The Cluster
To remove the cluster, update the cluster/installation.yaml to include `uninstalled: true`, 
then re-run the Gitlab pipeline or Porter installation command as in the Deployment section. The
cluster and all related resources will be destroyed.
