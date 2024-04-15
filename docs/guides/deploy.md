# Deploying the AWS Gateway API Controller

This Deployment Guide provides an end-to-end procedure to use the AWS Gateway API Controller with [Amazon Elastic Kubernetes Service](https://aws.amazon.com/eks/). 

The AWS Gateway API Controller can be used on any Kubernetes cluster on AWS. Check out the [Advanced Configurations](deploy.md/#advanced-configurations) section below for instructions on how to install and run the controller on self-hosted Kubernetes clusters on AWS.

## Deploy the controller on Amazon EKS

Amazon EKS is a simple, recommended way of preparing a cluster for running services with AWS Gateway API Controller. 

### Prerequisites

Install these tools before proceeding:

1. [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html),
2. `kubectl` - [the Kubernetes CLI](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/),
3. `helm` - [the package manager for Kubernetes](https://helm.sh/docs/intro/install/),
4. `eksctl`- [the CLI for Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/setting-up.html),
5. `jq` - [CLI to manipulate json files](https://jqlang.github.io/jq/).

### Setup

Set your AWS Region and Cluster Name as environment variables. See the [Amazon VPC Lattice FAQs](https://aws.amazon.com/vpc/lattice/faqs/) for a list of supported regions.
   ```bash linenums="1"
   export AWS_REGION=<cluster_region>
   export CLUSTER_NAME=<cluster_name>
   ```

**Create a cluster (optional)**

You can easily create a cluster with `eksctl`, the CLI for Amazon EKS:
   ```bash 
   eksctl create cluster --name $CLUSTER_NAME --region $AWS_REGION
   ```

**Allow traffic from Amazon VPC Lattice**

You must set up security groups so that they allow all Pods communicating with VPC Lattice to allow traffic from the VPC Lattice managed prefix lists.  See [Control traffic to resources using security groups](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html) for details. Lattice has both an IPv4 and IPv6 prefix lists available.

1. Configure the EKS nodes' security group to receive traffic from the VPC Lattice network. 

    ```bash 
    CLUSTER_SG=<your_node_security_group>
    ```
    !!!Note
        If you have created the cluster with `eksctl create cluster --name $CLUSTER_NAME --region $AWS_REGION` command, you can use this command to export the Security Group ID:

        ```bash 
        CLUSTER_SG=$(aws eks describe-cluster --name $CLUSTER_NAME --output json| jq -r '.cluster.resourcesVpcConfig.clusterSecurityGroupId')
        ```

    ```bash linenums="1"
    PREFIX_LIST_ID=$(aws ec2 describe-managed-prefix-lists --query "PrefixLists[?PrefixListName=="\'com.amazonaws.$AWS_REGION.vpc-lattice\'"].PrefixListId" | jq -r '.[]')
    aws ec2 authorize-security-group-ingress --group-id $CLUSTER_SG --ip-permissions "PrefixListIds=[{PrefixListId=${PREFIX_LIST_ID}}],IpProtocol=-1"
    PREFIX_LIST_ID_IPV6=$(aws ec2 describe-managed-prefix-lists --query "PrefixLists[?PrefixListName=="\'com.amazonaws.$AWS_REGION.ipv6.vpc-lattice\'"].PrefixListId" | jq -r '.[]')
    aws ec2 authorize-security-group-ingress --group-id $CLUSTER_SG --ip-permissions "PrefixListIds=[{PrefixListId=${PREFIX_LIST_ID_IPV6}}],IpProtocol=-1"
    ```

**Set up IAM permissions**

The AWS Gateway API Controller needs to have necessary permissions to operate.

1. Create a policy (`recommended-inline-policy.json`) in IAM with the following content that can invoke the Gateway API and copy the policy arn for later use:

    ```bash  linenums="1"
    curl https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/examples/recommended-inline-policy.json -o recommended-inline-policy.json
    
    aws iam create-policy \
        --policy-name VPCLatticeControllerIAMPolicy \
        --policy-document file://recommended-inline-policy.json

    export VPCLatticeControllerIAMPolicyArn=$(aws iam list-policies --query 'Policies[?PolicyName==`VPCLatticeControllerIAMPolicy`].Arn' --output text)
    ```

1. Create the `aws-application-networking-system` namespace:
```bash  
kubectl apply -f https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/deploy-namesystem.yaml
```

You can choose from [Pod Identities](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html) (recommended) and [IAM Roles For Service Accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) to set up controller permissions.

=== "Pod Identities"

    **Set up the Pod Identities Agent**

    To use Pod Identities, we need to [set up the Agent](https://docs.aws.amazon.com/eks/latest/userguide/pod-id-agent-setup.html) and to configure the controller's Kubernetes Service Account to assume necessary permissions with EKS Pod Identity.

    !!!Note "Read if you are using a custom node role"
        The node role needs to have permissions for the Pod Identity Agent to do the `AssumeRoleForPodIdentity` action in the EKS Auth API. Follow [the documentation](https://docs.aws.amazon.com/eks/latest/userguide/pod-id-agent-setup.html) if you are not using the AWS managed policy [AmazonEKSWorkerNodePolicy](https://docs.aws.amazon.com/eks/latest/userguide/security-iam-awsmanpol.html#security-iam-awsmanpol-AmazonEKSWorkerNodePolicy).


    1. Run the following AWS CLI command to create the Pod Identity addon.
    ```bash
    aws eks create-addon --cluster-name $CLUSTER_NAME --addon-name eks-pod-identity-agent --addon-version v1.0.0-eksbuild.1
    ```
    ```bash
    kubectl get pods -n kube-system | grep 'eks-pod-identity-agent'
    ```

    **Assign role to Service Account**

    Create an IAM role and associate it with a Kubernetes service account.

    1. Create a Service Account.

        ```bash
        cat >gateway-api-controller-service-account.yaml <<EOF
        apiVersion: v1
        kind: ServiceAccount
        metadata:
            name: gateway-api-controller
            namespace: aws-application-networking-system
        EOF
        kubectl apply -f gateway-api-controller-service-account.yaml
        ```

    1. Create a trust policy file for the IAM role.

        ```bash
        cat >trust-relationship.json <<EOF
        {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "AllowEksAuthToAssumeRoleForPodIdentity",
                    "Effect": "Allow",
                    "Principal": {
                        "Service": "pods.eks.amazonaws.com"
                    },
                    "Action": [
                        "sts:AssumeRole",
                        "sts:TagSession"
                    ]
                }
            ]
        }
        EOF
        ```

    1. Create the role.

        ```bash
        aws iam create-role --role-name VPCLatticeControllerIAMRole --assume-role-policy-document file://trust-relationship.json --description "IAM Role for AWS Gateway API Controller for VPC Lattice"
        aws iam attach-role-policy --role-name VPCLatticeControllerIAMRole --policy-arn=$VPCLatticeControllerIAMPolicyArn
        export VPCLatticeControllerIAMRoleArn=$(aws iam list-roles --query 'Roles[?RoleName==`VPCLatticeControllerIAMRole`].Arn' --output text)
        ```
    
    1. Create the association

        ```bash
        aws eks create-pod-identity-association --cluster-name $CLUSTER_NAME --role-arn $VPCLatticeControllerIAMRoleArn --namespace aws-application-networking-system --service-account gateway-api-controller
        ```

=== "IRSA"

    You can use AWS IAM Roles for Service Accounts (IRSA) to assign the Controller necessary permissions via a ServiceAccount.

    1. Create an IAM OIDC provider: See [Creating an IAM OIDC provider for your cluster](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html) for details.
        ```bash 
        eksctl utils associate-iam-oidc-provider --cluster $CLUSTER_NAME --approve --region $AWS_REGION
        ```

    1. Create an iamserviceaccount for pod level permission:

        ```bash  linenums="1"
        eksctl create iamserviceaccount \
            --cluster=$CLUSTER_NAME \
            --namespace=aws-application-networking-system \
            --name=gateway-api-controller \
            --attach-policy-arn=$VPCLatticeControllerIAMPolicyArn \
            --override-existing-serviceaccounts \
            --region $AWS_REGION \
            --approve
        ```

### Install the Controller

1. Run **either** `kubectl` or `helm` to deploy the controller. Check [Environment Variables](../guides/environment.md) for detailed explanation of each configuration option.

    === "Helm"

        ```bash  linenums="1"
        # login to ECR
        aws ecr-public get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin public.ecr.aws
        # Run helm with either install or upgrade
        helm install gateway-api-controller \
            oci://public.ecr.aws/aws-application-networking-k8s/aws-gateway-controller-chart \
            --version=v1.0.4 \
            --set=serviceAccount.create=false \
            --namespace aws-application-networking-system \
            --set=log.level=info # use "debug" for debug level logs
        ```

    === "Kubectl"

        ```bash 
        kubectl apply -f https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/deploy-v1.0.4.yaml
        ```


1. Create the `amazon-vpc-lattice` GatewayClass:
   ```bash 
   kubectl apply -f https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/gatewayclass.yaml
   ```

## Advanced configurations

The section below covers advanced configuration techniques for installing and using the AWS Gateway API Controller. This includes things such as running the controller on a self-hosted cluster on AWS or using an IPv6 EKS cluster.

### Using a self-managed Kubernetes cluster

You can install AWS Gateway API Controller to a self-managed Kubernetes cluster in AWS.

However, the controller utilizes [IMDS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html) to get necessary information from instance metadata, such as AWS account ID and VPC ID. So:

- **If your cluster is using IMDSv2.** ensure the hop limit is 2 or higher to allow the access from the controller:

    ```bash
    aws ec2 modify-instance-metadata-options --http-put-response-hop-limit 2 --region <region> --instance-id <instance-id>
    ```

- **If your cluster cannot access to IMDS.** ensure to specify the[configuration variables](environment.md) when installing the controller.

### IPv6 support

IPv6 address type is automatically used for your services and pods if
[your cluster is configured to use IPv6 addresses](https://docs.aws.amazon.com/eks/latest/userguide/cni-ipv6.html).

If your cluster is configured to be dual-stack, you can set the IP address type
of your service using the `ipFamilies` field. For example:

```yaml title="parking_service.yaml"
apiVersion: v1
kind: Service
metadata:
  name: ipv4-target-in-dual-stack-cluster
spec:
  ipFamilies:
    - "IPv4"
  selector:
    app: parking
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8090
```



