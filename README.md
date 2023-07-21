# Package and deploy Kubernetes resources on IBM IKS cluster using helm chart repository in IBM COS

This pattern helps you to package Kubernetes resources together and manage Kubernetes applications efficiently, regardless of their complexity. The pattern helps to integrate Helm into your existing continuous integration and continuous delivery (CI/CD) pipelines to deploy applications into an IBM Kubernetes cluster. Helm is a Kubernetes package manager that helps you manage Kubernetes applications. Helm charts help to define, install, and upgrade complex Kubernetes applications. Charts can be versioned and stored in Helm repositories, which improves mean time to restore (MTTR) during outages.

## Prerequisites

- An IBM IKS cluster
- [IBMCLOUD CLI](https://cloud.ibm.com/docs/cli?topic=cli-install-ibmcloud-cli)
- [Kubectl](https://kubernetes.io/docs/tasks/tools/) for configuring the IBM IKS kubeconfig file for the target cluster in the client machine
- [Helm client](https://helm.sh/docs/intro/install/)
- Bucket in IBM COS to setup helm repository

## Steps

- Log in to IBM Cloud and download the kubeconfig of the IBM IKS cluster
  ```
  ibmcloud ks cluster config --admin -c <cluster_name_or_id>
  ```
- Validate that Helm is able to communicate with the Kubernetes API server within the IBM IKS cluster.
  ```
  helm version
  ```
- Create a helm chart named `my-nginx` on the client machine.
  ```
  helm create my-nginx
  ```
- Validate the chart for any syntactical error before installing it in the target cluster.
  ```
  helm lint my-nginx
  ```
- Create a namespace.
  ```
  kubectl create ns test-helm
  ```
- Run the Helm chart installation on the cluster.
  ```
  helm install my-nginx-release --debug my-nginx/ --namespace test-helm
  ```
  The `namespace` flag specifies the namespace in which the resources part of this chart will be created.

- Review the resources that were created as part of the Helm chart in the `test-helm` namespace.
  ```
  kubectl get all -n test-helm
  ```
- Create a directory called `charts` on the other client machine. The example in this pattern uses `s3://<bucket_name>/charts` as the target chart repository.
  ```
  mkdir charts
  ```
- Install the `helm-s3` plugin on your client machine for connecting to IBM COS.
  ```
  helm plugin install --version 0.10.0 https://github.com/hypnoglow/helm-s3.git
  ```
- Set the following required environment variables required by `helm-s3` plugin to connect to bucket on IBM COS which would be used for hosting helm repository.
  ```
  export AWS_ACCESS_KEY_ID=<AWS_ACCESS_KEY_ID>
  export AWS_SECRET_ACCESS_KEY=<AWS_SECRET_ACCESS_KEY>
  export AWS_DEFAULT_REGION=<AWS_DEFAULT_REGION>
  export AWS_DEFAULT_ENDPOINT=<AWS_DEFAULT_ENDPOINT>
  ```
- Initialize the target folder `charts` as a Helm repository under bucket `helm-charts-bucket`. 
  ```
  helm s3 init s3://helm-charts-bucket/charts
  ```
  The command creates an `index.yaml` file in the target to track all the chart information that is stored at that location.
  
- Add the repository in the client machine.
  ```
  helm repo add my-helm-charts s3://helm-charts-bucket/charts
  ```
- To view the list of repositories in the Helm client machine, run
  ```
  helm repo ls
  ```
- To package the `my-nginx` chart that you created, run 
  ```
  helm package ./my-nginx/
  ```
  The command packages all the contents of the `my-nginx` chart folder into an archive file, which is named using the version number that is mentioned in the Chart.yaml file.

- To upload the package to the Helm repository , run the following command, using the correct name of the `.tgz` file generated in previous step.
  ```
  helm s3 push ./my-nginx-0.1.0.tgz my-helm-charts
  ```
- To confirm that the chart appears both locally and in the Helm repository in IBM COS, run the following command.
  ```
  helm search repo my-helm-charts
  ```
- To verify helm chart upgrade, you can modify the helm chart. To modify the chart, in `values.yaml`, change the `replicaCount` value to 2. Also in `Chart.yaml`, change `version` to `0.1.1`. Then re-package the chart again.
  ```
  helm package ./my-nginx/
  ```
  This will generate a file named `my-nginx-0.1.1.tgz`.
  
- Upload new package to the Helm repository.
  ```
  helm s3 push ./my-nginx-0.1.1.tgz my-helm-charts
  ```
- To view all the available versions of a chart, run the following command with the `--versions` flag. Without the flag, Helm by default displays the latest uploaded version of a chart.
  ```
  helm search repo my-nginx --versions
  ```
- The search results from the previous step show the multiple versions of the `my-nginx` chart. To upgrade installed release with the new version (0.1.1) from the IBM COS Helm repository, use the following command.
  ```
  helm upgrade my-nginx-release my-helm-charts/my-nginx --version 0.1.1 --namespace test-helm
  ```
- To list all the revisions for a specific release that have been installed using Helm, run the following command.
  ```
  helm history my-nginx-release
  ```
- Before switching or rolling back to a working version, and for an additional layer of validation before installing a revision, view which values were passed to each of the revisions by using the following command.
  ```
  helm get --revision=2 my-nginx-release
  ```














