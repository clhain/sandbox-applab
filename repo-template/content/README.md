# ${{ values.name }} Readme
Describe your application here.

## Deploy The Cluster

### Using GitLab Runner
The deployment pipeline can automatically deploy and update the cluster once the following variables are set:

1. Add a mongodb connection string variable for porter to maintain state between runs:
   * GitLab: Settings -> CI/CD -> Variables -> Add key **PORTER_MONGO_CONNECTION_STRING** and value (your string e.g. mongodb+srv://USER_HERE:PASSWORD_HERE@YOUR_HOST/?retryWrites=true&w=majority)
2. Add the BASE64 encoded copy of your deployer service account key:
   * GitLab: Settings -> CI/CD -> Variables -> Add key **BASE64_GOOGLE_CREDENTIALS** and value (cat YOUR_GLOUD_KEY | base64 -w0)
3. Add the OIDC Client Secret (or empty string)
   * GitLab: Settings -> CI/CD -> Variables -> Add key **OIDC_CLIENT_SECRET** and value (your client secret or "")

Manually trigger the pipeline, and set variable **"UPDATE_CLUSTER"** to "true" from CI/CD -> Pipelines -> Run Pipeline

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

## Authenticate The Cluster With GitLab
In Gitlab, go to Infrastructure -> Kubernetes Cluster -> Connect a cluster, then select or create an agent token.


## Deploy Code To The Cluster

### With AutoDevops
Once the cluster is initialized, you can authorize it for use with auto-devops.

### Use With ArgoCd

trigger porter install / upgrade
