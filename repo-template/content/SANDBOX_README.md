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

### Using GitLab Runner
The deployment pipeline can automatically deploy and update the cluster once the following variables are set:

1. Add a mongodb connection string variable for porter to maintain state between runs:
   * GitLab: Settings -> CI/CD -> Variables -> Add key **PORTER_MONGO_CONNECTION_STRING** and value (your string e.g. mongodb+srv://USER_HERE:PASSWORD_HERE@YOUR_HOST/?retryWrites=true&w=majority)
2. Add the BASE64 encoded copy of your deployer service account key:
   * GitLab: Settings -> CI/CD -> Variables -> Add key **BASE64_GOOGLE_CREDENTIALS** and value (cat YOUR_GLOUD_KEY | base64 -w0)
3. Add the OIDC Client Secret (or empty string)
   * GitLab: Settings -> CI/CD -> Variables -> Add key **OIDC_CLIENT_SECRET** and value (your client secret or "")

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
[Connect to the cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl) using the gcloud connection string from the gcloud portal.

Follow the [Verification](https://clhain.github.io/sandbox/installation/quick-start/#verification) section of the sandbox
documentation to ensure the deployment rolls out. You'll need to add the Gitlab Agent token (see the next section) before
the gitlab-agent app will enter the Healthy state.

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
Once the cluster is initialized, you can authorize it for use with auto-devops.

### Use With ArgoCd

trigger porter install / upgrade


### Uninstall The Cluster
To remove the cluster, update the cluster/installation.yaml to include `uninstalled: true`, 
then re-run the Gitlab pipeline or Porter installation command as in the Deployment section. The
cluster and all related resources will be destroyed.
