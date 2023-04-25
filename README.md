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
