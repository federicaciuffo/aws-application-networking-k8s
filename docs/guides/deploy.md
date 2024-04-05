# Deploying the AWS Gateway API Controller

This Deployment Guide provides an end-to-end procedure to use the AWS Gateway API Controller with [Amazon Elastic Kubernetes Service](https://aws.amazon.com/eks/). 

The AWS Gateway API Controller can be used on any Kubernetes cluster on AWS. Check out the [Advanced Configurations](deploy.md/#advanced-configurations) section below for instructions on how to install and run the controller on self-hosted Kubernetes clusters on AWS.

## Deploy the controller on Amazon EKS

Amazon EKS is a simple, recommended way of preparing a cluster for running services with AWS Gateway API Controller. The AWS Gateway API Controller can be installed with either `helm` or `kubectl`. Additionally, it requires cloud provider permissions for VPC Lattice, for AWS IAM Roles for Service Accounts (IRSA) should be used. IRSA permits the Controller (within the cluster) to make privileged requests to AWS (as the cloud provider) via a ServiceAccount.

### Prerequisites

Install these tools before proceeding:

1. [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html),
2. `kubectl` - [the Kubernetes CLI](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/),
3. `helm` - [the package manager for Kubernetes](https://helm.sh/docs/intro/install/),
4. `eksctl`- [the CLI for Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/setting-up.html).

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

1. Create an IAM OIDC provider: See [Creating an IAM OIDC provider for your cluster](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html) for details.
    ```bash 
    eksctl utils associate-iam-oidc-provider --cluster $CLUSTER_NAME --approve --region $AWS_REGION
    ```
1. Create a policy (`recommended-inline-policy.json`) in IAM with the following content that can invoke the gateway API and copy the policy arn for later use:

    ```bash  linenums="1"
    cat <<EOF > recommended-inline-policy.json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "vpc-lattice:*",
                    "ec2:DescribeVpcs",
                    "ec2:DescribeSubnets",
                    "ec2:DescribeTags",
                    "ec2:DescribeSecurityGroups",
                    "logs:CreateLogDelivery",
                    "logs:GetLogDelivery",
                    "logs:DescribeLogGroups",
                    "logs:PutResourcePolicy",
                    "logs:DescribeResourcePolicies",
                    "logs:UpdateLogDelivery",
                    "logs:DeleteLogDelivery",
                    "logs:ListLogDeliveries",
                    "tag:GetResources",
                    "firehose:TagDeliveryStream",
                    "s3:GetBucketPolicy",
                    "s3:PutBucketPolicy"
                ],
                "Resource": "*"
            },
            {
                "Effect" : "Allow",
                "Action" : "iam:CreateServiceLinkedRole",
                "Resource" : "arn:aws:iam::*:role/aws-service-role/vpc-lattice.amazonaws.com/AWSServiceRoleForVpcLattice",
                "Condition" : {
                    "StringLike" : {
                        "iam:AWSServiceName" : "vpc-lattice.amazonaws.com"
                    }
                }
            },
            {
                "Effect" : "Allow",
                "Action" : "iam:CreateServiceLinkedRole",
                "Resource" : "arn:aws:iam::*:role/aws-service-role/delivery.logs.amazonaws.com/AWSServiceRoleForLogDelivery",
                "Condition" : {
                    "StringLike" : {
                        "iam:AWSServiceName" : "delivery.logs.amazonaws.com"
                    }
                }
            }
        ]
    }
    EOF
    ```

    ```bash  linenums="1"
    aws iam create-policy \
        --policy-name VPCLatticeControllerIAMPolicy \
        --policy-document file://recommended-inline-policy.json
    ```

1. Create the `aws-application-networking-system` namespace:
   ```bash  
   kubectl apply -f https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/examples/deploy-namesystem.yaml
   ```
1. Retrieve the policy ARN:
   ```bash  
   export VPCLatticeControllerIAMPolicyArn=$(aws iam list-policies --query 'Policies[?PolicyName==`VPCLatticeControllerIAMPolicy`].Arn' --output text)
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
        kubectl apply -f https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/examples/deploy-v1.0.4.yaml
        ```


1. Create the `amazon-vpc-lattice` GatewayClass:
   ```bash 
   kubectl apply -f https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/examples/gatewayclass.yaml
   ```

1. You are all set! Check our [Getting Started Guide](getstarted.md) to try setting up service-to-service communication.

## Advanced configurations

The section below covers advanced configuration techniques for installing and using the AWS Gateway API Controller. This includes things such as running the controller on a self-hosted cluster on AWS or using an IPv6 EKS cluster.

### Using a self-managed Kubernetes cluster

You can install AWS Gateway API Controller to a self-managed Kubernetes cluster in AWS.

The controller utilizes [IMDS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html) to get necessary information from instance metadata, such as AWS account ID and VPC ID.
If your cluster is using IMDSv2, ensure the hop limit is 2 or higher to allow the access from the controller:

```bash linenums="1"
aws ec2 modify-instance-metadata-options --http-put-response-hop-limit 2 --region <region> --instance-id <instance-id>
```

Alternatively, you can manually provide configuration variables when installing the controller.

### IPv6 support

IPv6 address type is automatically used for your services and pods if
[your cluster is configured to use IPv6 addresses](https://docs.aws.amazon.com/eks/latest/userguide/cni-ipv6.html).

```bash linenums="1"
# To create an IPv6 cluster
eksctl create cluster -f https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/examples/ipv6-cluster.yaml
```

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



