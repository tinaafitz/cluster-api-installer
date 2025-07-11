# Creating a ROSA HCP cluster

## Prerequisites

1. Complete the prerequisite actions listed in the [Set up to use the ROSA CLI](https://docs.aws.amazon.com/rosa/latest/userguide/set-up.html) documentation.

2. Create an Amazon VPC that will be used with ROSA HCP using the Terraform template as follows:
    1. Install the Terraform CLI. For more information, see the installation instructions in the [Terraform documentation](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli).
	1. Open a terminal session and clone the Terraform VPC repository.

	   `git clone https://github.com/openshift-cs/terraform-vpc-example`
	
	1. Follow the commands below to create the VPC:
	   ```shell
	   cd terraform-vpc-example
	   terraform init
	   terraform plan -out rosa.tfplan -var region=<region>
	   terraform apply rosa.tfplan
	   ```
       The subnet-ids will be displayed once Terraform has successfully created the VPC.	   
       ```     
       private-subnet-id: subnet-0889990000000000   
       public-subnet-id: subnet-054ad00000000000
       ...
	   ```
3. Create the required IAM roles and OpenID Connect configuration.
   
    1. Create the required IAM account roles and policies.
       ```     
        rosa create account-roles --force-policy-creation
       ```
    2. Create the OpenID Connect (OIDC) configuration.

       ```     
       rosa create oidc-config --mode=auto
       ```
    3. Copy the OIDC config ID <OIDC_CONFIG_ID> provided in the ROSA CLI output. 
      
       The OIDC config ID needs to be provided later to create the ROSA HCP cluster.  
       You can list the available OIDC config IDs using the following command:

       ```     
       rosa list oidc-config
       ```

   4. Create the required IAM operator roles.   
   You must supply a prefix in <PREFIX_NAME> and replace the <OIDC_CONFIG_ID> with the OIDC config ID copied previously.

      ```     
       rosa create operator-roles --prefix <PREFIX_NAME> --oidc-config-id <OIDC_CONFIG_ID> --hosted-cp
      ```
     
   5. Verify the IAM operator roles were created. 
      ```     
      rosa list operator-roles
      ```

4. Assuming you have an OpenShift cluster v4.16 or later running, install ACM (Advanced Cluster Management) operator v2.13 from the [operator hub](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.13/html/install/installing#installing-from-the-operatorhub) or using OpenShift Container Platform [CLI](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.13/html/install/installing#installing-from-the-cli)

Notes: The steps for creating the AWS VPC and IAM roles will be declarative through CAPA custom resources next release.

## Enable CAPI, CAPA and auto import 

The MultiClusterEngine custom resource CAPI & CAPA features must be enabled once the ACM operator installation and pre-request steps have been completed. 

1. Verify the default MultiClusterEngine CR has been created. 

```shell
 oc get multiclusterengine multiclusterengine 
 NAME     STATUS      AGE   CURRENTVERSION   DESIREDVERSION
 multiclusterengine   Available   11d   2.8.0            2.8.0
```

2. Edit the MultiClusterEngine engine.

`oc edit multiclusterengine multiclusterengine`

Modify the components list cluster-api-preview & cluster-api-provider-aws-preview items as shown below:

```yaml
    - configOverrides: {}
      enabled: true
      name: cluster-api-preview
    - configOverrides: {}
      enabled: true
      name: cluster-api-provider-aws-preview

```

The changes are automatically saved. 

3. Verify the CAPI & CAPA deployments were installed.

```shell
oc get deploy -n multicluster-engine
NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
capa-controller-manager               1/1     1            1           12d
capi-controller-manager               1/1     1            1           12d
```

4. Verify the ClusterManager CR cluster-manager was created.
```
oc get ClusterManager
NAME              AGE
cluster-manager   12d
```

5. Edit the cluster-manager to enable auto import.

`oc edit ClusterManager cluster-manager`

Add the registrationConfiguration section as shown below:

```yaml
apiVersion: operator.open-cluster-management.io/v1
kind: ClusterManager
metadata:
  name: cluster-manager
spec:
  registrationConfiguration:
    featureGates:
    - feature: ClusterImporter
      mode: Enable
    - feature: ManagedClusterAutoApproval
      mode: Enable
    autoApproveUsers:
    - system:serviceaccount:multicluster-engine:agent-registration-bootstrap
```

Bind the CAPI manager permission to the import controller by applying the ClusterRoleBinding below.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-manager-registration-capi
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: capi-operator-manager-role
subjects:
- kind: ServiceAccount
  name: registration-controller-sa
  namespace: open-cluster-management-hub
```

## Permissions
### AWS credentials

1. Set the AWS credentials secret. Run the command below after setting the AWS values and region.

``` echo '[default]
aws_access_key_id = <your-access-key>
aws_secret_access_key = <your-secret-access-key>
region = us-east-1
' | base64 -w 0
```

If you are using Multi-Factor Auth with AWS use the below command instead with the session token.

``` echo '[default]
aws_access_key_id = <your-access-key>
aws_secret_access_key = <your-secret-access-key>
aws_session_token= <your-aws-session-token>
region = us-east-1
' | base64 -w 0
```

2. Edit the capa-manager-bootstrap-credentials secret.   
Copy the output of the previous command and add it to the capa-manager-bootstrap-credentials secret.

`oc edit secret -n multicluster-engine capa-manager-bootstrap-credentials`

Make the changes to the data->credentials field as shown below:

```yaml
apiVersion: v1
data:
  credentials: <REPLACE_WITH_AWS_CREDENTIALS>
kind: Secret
metadata:
  name: capa-manager-bootstrap-credentials
  namespace: multicluster-engine
```

3. Restart the capa-controller-manager deployment.

`oc rollout restart deployment capa-controller-manager -n multicluster-engine`

### OCM Authentication using offline token OR service account credentials
CAPA controller requires Redhat OCM credentials to provision ROSA HCP. You can obtain OCM credentials in two ways;

1. Obtain an offline token by visiting https://console.redhat.com/openshift/token. Create a credentials secret within the target namespace with the token to be referenced later by `ROSAControlPlane`
    ```shell
        kubectl create secret generic rosa-creds-secret \
            --from-literal=ocmToken='eyJhbGciOiJIUzI1NiIsI....' \
        --from-literal=ocmApiUrl='https://api.openshift.com'
    ```
   Note: You can change the secret namespace similar to the `ROSAControlPlane` that will be created later.

OR

2. Create a service account by visiting [https://console.redhat.com/iam/service-accounts](https://console.redhat.com/iam/service-accounts). If you already have a service account, you can skip this step.

   For every newly created service account, make sure to activate the account using the [ROSA command line tool](https://github.com/openshift/rosa). First, log in using your newly created service account
   ```shell
   rosa login --client-id ... --client-secret ...
   ```
   Then activate your service account
   ```shell
   rosa whoami
   ```

   Create a new kubernetes secret with the service account credentials in the target namespace to be referenced later by `ROSAControlPlane`
    ```shell
    kubectl create secret generic rosa-creds-secret \
      --from-literal=ocmClientID='....' \
      --from-literal=ocmClientSecret='eyJhbGciOiJIUzI1NiIsI....' \
      --from-literal=ocmApiUrl='https://api.openshift.com'
    ```
    Note: to consume the secret without the need to reference it from your `ROSAControlPlane`, name your secret as `rosa-creds-secret` and create it in the CAPA manager namespace.
    ```shell
    kubectl -n multicluster-engine create secret generic rosa-creds-secret \
      --from-literal=ocmClientID='....' \
      --from-literal=ocmClientSecret='eyJhbGciOiJIUzI1NiIsI....' \
      --from-literal=ocmApiUrl='https://api.openshift.com'
    ```

## Creating the ROSA-HCP cluster

1. Apply the AWSClusterControllerIdentity below using `oc apply` command.

    ```yaml
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
    kind: AWSClusterControllerIdentity
    metadata:
      name: "default"
    spec:
      allowedNamespaces: {}  # matches all namespaces
    ```

2. Update the `ROSAControlPlane` template below with relative info created in the prerequisite steps then apply it using `oc apply ` command.

	```yaml
	apiVersion: v1
	kind: Namespace
	metadata:
	  name: ns-rosa-hcp
	---
	apiVersion: cluster.open-cluster-management.io/v1
	kind: ManagedCluster
	metadata:
	  name: rosa-hcp-1
	spec:
	  hubAcceptsClient: true
	---
	apiVersion: cluster.x-k8s.io/v1beta1
	kind: Cluster
	metadata:
	  name: "rosa-hcp-1"
	  namespace: "ns-rosa-hcp"
	spec:
	  clusterNetwork:
	    pods:
	      cidrBlocks: ["192.168.0.0/16"]
	  infrastructureRef:
	    apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
	    kind: ROSACluster
	    name: "rosa-hcp-1"
	    namespace: "ns-rosa-hcp"
	  controlPlaneRef:
	    apiVersion: controlplane.cluster.x-k8s.io/v1beta2
	    kind: ROSAControlPlane
	    name: "rosa-cp-1"
	    namespace: "ns-rosa-hcp"
	---
	apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
	kind: ROSACluster
	metadata:
	  name: "rosa-hcp-1"
	  namespace: "ns-rosa-hcp"
	spec: {}
	---
	apiVersion: controlplane.cluster.x-k8s.io/v1beta2
	kind: ROSAControlPlane
	metadata:
	  name: "rosa-cp-1"
	  namespace: "ns-rosa-hcp"
	spec:
	  credentialsSecretRef:
	    name: rosa-creds-secret
	  rosaClusterName: rosa-hcp-1
	  domainPrefix: rosa-hcp
	  version: "4.18.1"
	  ## The region should match the aws region used to create the VPC and subnets
	  region: "us-west-2"

	  ## Replace the IAM account roles below with the IAM roles created in the prerequisite steps
	  ## List the IAM account roles using command 'rosa list account-roles'
	  installerRoleARN: "arn:aws:iam:: 12345678910:role/your-prefix-HCP-ROSA-Installer-Role"
	  supportRoleARN: "arn:aws:iam:: 12345678910:role/your-prefix-HCP-ROSA-Support-Role"
	  workerRoleARN: "arn:aws:iam:: 12345678910:role/your-prefix-HCP-ROSA-Worker-Role"

	  ## Replace the oidc config below with the oidc config created in the prerequisite steps
	  ## List the oidc config using command `rosa list oidc-providers`
	  oidcID: "oidc-config-id"

	  ## Replace IAM operator roles below with the IAM roles created in the prerequisite steps
	  ## List the operator roles using command `rosa list operator-roles --prefix your-prefix`
	  rolesRef:
	    ingressARN: "arn:aws:iam::12345678910:role/your-prefix-openshift-ingress-operator-cloud-credentials"
	    imageRegistryARN: "arn:aws:iam::12345678910:role/your-prefix-openshift-image-registry-installer-cloud-credentials"
	    storageARN: "arn:aws:iam::12345678910:role/your-prefix-openshift-cluster-csi-drivers-ebs-cloud-credentials"
	    networkARN: "arn:aws:iam::12345678910:role/your-prefix-openshift-cloud-network-config-controller-cloud-credentials"
	    kubeCloudControllerARN: "arn:aws:iam::12345678910:role/your-prefix-kube-system-kube-controller-manager"
	    nodePoolManagementARN: "arn:aws:iam::12345678910:role/your-prefix-kube-system-capa-controller-manager"
	    controlPlaneOperatorARN: "arn:aws:iam::12345678910:role/your-prefix-kube-system-control-plane-operator"
	    kmsProviderARN: "arn:aws:iam::12345678910:role/your-prefix-kube-system-kms-provider"

	  ## Replace the subnets and availabilityZones with the subnets created in the prerequisite steps
	  subnets:
	    - "subnet-id"
	    - "subnet-id"
	  availabilityZones:
	    - az-1 # ex "us-west-2b"
	  network:
	    machineCIDR: "10.0.0.0/16"
	    podCIDR: "10.128.0.0/14"
	    serviceCIDR: "172.30.0.0/16"
	  defaultMachinePoolSpec:
	    instanceType: "m5.xlarge"
	    autoscaling:
	      maxReplicas: 3
	      minReplicas: 2
	  additionalTags:
	    env: "demo"
	    profile: "hcp"
	```
3. Check the ROSAControlPlane status.

`oc get ROSAControlPlane rosa-cp-1 -n ns-rosa-hcp -o yaml`

The ROSA HCP cluster should take around 40 minutes to be fully provisioned.

## Support

When creating an issue for ROSA HCP cluster, please make sure to include logs for the capa-controller-manager and capi-controller-manager deployment pods. 
The logs can be saved to text file using the commands below:

```shell
$ kubectl get pod -n multicluster-engine
NAME                                      READY   STATUS    RESTARTS   AGE
capa-controller-manager-77f5b946b-sddcg   1/1     Running   1          3d3h

$ kubectl logs -n multicluster-engine capa-controller-manager-77f5b946b-sddcg > capa-controller-manager-logs.txt

$ kubectl get pod -n multicluster-engine
NAME                                       READY   STATUS    RESTARTS   AGE
capi-controller-manager-78dc897784-f8gpn   1/1     Running   18         26d

$ kubectl logs -n multicluster-engine capi-controller-manager-78dc897784-f8gpn > capi-controller-manager-logs.txt
```

 Make sure to also include yaml files for all the resources used to create the ROSA HCP cluster:
- `Cluster`
- `ROSAControlPlane`
- `MachinePool`
- `ROSAMachinePool`
