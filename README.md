## Deployment
To deploy the helm chart to a cluster:
* Ensure the following are installed:
  * [helm3](https://helm.sh/docs/intro/install/)
  * [gcloud](https://cloud.google.com/sdk/docs/install)
  * [gke-gcloud-auth-plugin](https://cloud.google.com/blog/products/containers-kubernetes/kubectl-auth-changes-in-gke)
  * [helm-secrets plugin](https://github.com/jkroepke/helm-secrets/wiki/Installation)
    - Run `helm plugin install https://github.com/jkroepke/helm-secrets --version v4.4.2`
      - Upgrade to newer version by first uninstalling plugin `helm plugin uninstall secrets` then install as above
    - Install `helm-secrets` backend [vals](https://github.com/variantdev/vals/releases/latest) in your PATH
* Ensure your local kubectl is pointing at the correct cluster: `kubectl config get-contexts`
  * If on `datasembly-dev`, use the service account for the cluster
    * Download the [key file](https://datasembly.1password.com/vaults/nyyd4ii4vpg2m2ubqyyuchmp3e/allitems/ce5jdrys5cfzzcbq5crmf62uh4)
    * `gcloud auth activate-service-account --project=datasembly-dev --key-file=/path/to/frontend-sa-datasembly-dev-key.json`
  * Refresh gcloud credentials `gcloud container clusters get-credentials frontend-cluster --region us-east1-b`
* Make sure you are in the `frontend-cluster` directory: `cd deployment/frontend-cluster`
  * Run `helm dep update`
    * Run if setting up for the first time:
      ```shell
      helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
      helm repo add jetstack https://charts.jetstack.io 
      helm dependency update
      ```
  note: We should probably have cert manager as a dependency of this deployment
* Do a DRY RUN of the deployment to ensure everything works:
  
  ```bash
  HELM_SECRETS_BACKEND=vals
  helm secrets -b vals decrypt <your/path/to/>secrets-[ENV].yaml | tee <your/path/to/>secrets-[ENV].yaml.dec
  helm upgrade --atomic -f <your/path/to/>values-[ENV].yaml -f <your/path/to/>secrets-[ENV].yaml.dec default . --debug --dry-run > <your-text-file-updated-dry-run-manifest.txt> 2>&1
  ```
  * ENV will be either `dev` or `prod` (we only have `dev` currently)
  * `default` is the name of the deployment
  * `.` is the path to the helm chart you are installing
  * If this is a brand-new cluster, use `install` instead of `upgrade`
* If the dry run looks correct, you can run the command again without the dry-run and debug flags

### When should I do a deployment?
You should do a deployment anytime you need to change the configuration of one of our services.  This includes things like:
* Resources: upgrading the memory of a service
* Replicas: How many pods should be running
* Networking configurations: Ex. adding a new service to our package and creating a new URL for it
* Environment Variables: All of our services rely on environment variables; any change will require a new deployment
* Dependency updates: Bump version numbers for the charts referenced in [Chart.yaml](./Chart.yaml) and [values.yaml](./values.yaml),
then run `helm dependency update` to update [Chart.lock](./Chart.lock).
  * [Chart.yaml](./Chart.yaml):
    * ingress-nginx: https://github.com/kubernetes/ingress-nginx/releases?q=%22helm-chart%22&expanded=true
    * cert-manager: https://charts.jetstack.io
  * [values.yaml](./values.yaml):
    * cloudsqlproxy: https://console.cloud.google.com/gcr/images/cloudsql-docker/global/gce-proxy

## Rolling back
Sometimes we push code that makes our app unusable.  If this happens in our dev environment, not to worry, we should focus on fixing the code and pushing the fix.  
If we notice that our prod environment is broken, then our rollback plan is easy:
* Identify which service if broken, and figure out the previous version of that service
* Update the `imageTag` property in the corresponding `values-[ENV].yaml` file
* Deploy the helm chart to the proper cluster.  The broken image should be rolled back to the previous good image

Remember to use the `--atomic` flag when performing Helm upgrades to automatically roll back changes on a failed deployment.
