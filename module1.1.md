### Enable EKS Auto Mode

Choose your preferred method to enable Amazon EKS Auto Mode in the cluster used for this workshop.

This section describes how to enable Amazon EKS Auto Mode on your existing Amazon EKS clusters using AWS CLI.
1. Update the cluster IAM Role

EKS Auto Mode docs 

describe IAM policies to be added to the cluster role, in addition to the already existing AmazonEKSClusterPolicy:

    AmazonEKSComputePolicy 

AmazonEKSBlockStoragePolicy 
AmazonEKSNetworkingPolicy 
AmazonEKSLoadBalancingPolicy 

➤ We can either add these policies manually, through the AWS IAM console, or use the following command:

for POLICY in \
  "arn:aws:iam::aws:policy/AmazonEKSComputePolicy" \
  "arn:aws:iam::aws:policy/AmazonEKSBlockStoragePolicy" \
  "arn:aws:iam::aws:policy/AmazonEKSLoadBalancingPolicy" \
  "arn:aws:iam::aws:policy/AmazonEKSNetworkingPolicy" \
  "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
do
  echo "Attaching policy ${POLICY} to IAM role ${DEMO_CLUSTER_ROLE_NAME}..."
  aws iam attach-role-policy --role-name ${DEMO_CLUSTER_ROLE_NAME} --policy-arn ${POLICY}
done

In addition, the trust policy of the cluster role needs to be changed to allow EKS to automate routine tasks.

➤ Let's execute the following command to add the tagSession to the trust policy:

aws iam update-assume-role-policy --role-name $DEMO_CLUSTER_ROLE_NAME --policy-document '{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": [
        "sts:AssumeRole",
        "sts:TagSession"
      ]
    }
  ]
}'

➤ Verify that the IAM role has been updated properly:

aws iam get-role --role-name ${DEMO_CLUSTER_ROLE_NAME} | \
  jq -r '.Role.AssumeRolePolicyDocument.Statement[].Action[]'

aws iam list-attached-role-policies --role-name ${DEMO_CLUSTER_ROLE_NAME} | \
  jq -r '.AttachedPolicies[].PolicyName'

The output should be (note the AmazonEKSClusterPolicy that the role already had attached before):

sts:AssumeRole
sts:TagSession
AmazonEKSClusterPolicy
AmazonEKSNetworkingPolicy
AmazonEKSComputePolicy
AmazonEKSBlockStoragePolicy
AmazonEKSLoadBalancingPolicy

2. Enable EKS Auto Mode

➤ We can now run the following command to enable EKS Auto Mode:

aws eks update-cluster-config \
    --name ${DEMO_CLUSTER_NAME} \
    --compute-config enabled=true,nodeRoleArn=${DEMO_CLUSTER_NODE_ROLE_ARN},nodePools=system,general-purpose \
    --kubernetes-network-config '{"elasticLoadBalancing":{"enabled": true}}' \
    --storage-config '{"blockStorage":{"enabled": true}}'

This should produce an output similar to the following (note the status being InProgress):

{
    "update": {
        "id": "...,
        "status": "InProgress",
        "type": "AutoModeUpdate",
        "params": [
            ...
        ],
        "createdAt": "...",
        "errors": []
    }
}

Note

The compute, block storage, and load balancing capabilities in the command above must all be enabled or disabled in the same request.

In addition to enabling these capabilities, we've also configured the node IAM role to be attached to EKS Auto Mode managed instances, and defined the two built-in 

EKS Auto Mode NodePools. The system NodePool is intended for cluster-critical applications, and the general-purpose NodePool is intended for general purpose workloads in the cluster.

It might take couple of seconds for the cluster to get the API call from the CLI and reach an "Updating" state. If any of the below commands shows that the cluster is "Active", please wait a couple of seconds, and run it again.

➤ We can now check the upgrade status of the cluster using the following command:

aws eks describe-cluster --name ${DEMO_CLUSTER_NAME} --query 'cluster.status'

➤ Alternatively, we can run the following command to wait for the cluster to be in the Active state:

aws eks wait cluster-active --name ${DEMO_CLUSTER_NAME}

Verify EKS Auto Mode

➤ Once EKS Auto Mode has been enabled, we can verify the cluster state is now Active:

aws eks describe-cluster --name ${DEMO_CLUSTER_NAME} --query 'cluster.status'

➤ We can also check that new CRDs (that configure EKS Auto Mode capabilities) are installed in the cluster by running the following command. These CRDs are created and managed by EKS Auto Mode.

kubectl get crd | grep eks.amazonaws.com

The output should be similar to this (filtered to only the CRDs provided by EKS Auto Mode):

cninodes.eks.amazonaws.com                   2025-01-22T12:25:06Z
ingressclassparams.eks.amazonaws.com         2025-01-22T12:25:02Z
nodeclasses.eks.amazonaws.com                2025-01-22T12:25:02Z
nodediagnostics.eks.amazonaws.com            2025-01-22T12:25:02Z
targetgroupbindings.eks.amazonaws.com        2025-01-22T12:25:02Z

Summary

In this section we learned how to enable Amazon EKS Auto Mode.

We can now move on to the next section to deploy a sample application into our cluster.





