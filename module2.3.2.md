Advanced Networking

Overview | Handle IP exhaustion | Isolate Pod Network | Summary
Overview

Amazon EKS Auto Mode simplifies and automates critical networking tasks for pod and service connectivity by managing the VPC Container Network Interface (CNI) configuration and load balancer provisioning for the cluster.

In modern environments there are several additional use cases that require advanced network configuration.
IP exhaustion

By default, Amazon VPC CNI will assign pods an IP address selected from the primary subnet. The primary subnet is the subnet CIDR that the primary ENI is attached to, usually the subnet of the node/host.

If the subnet CIDR is too small, the CNI may not be able to acquire enough secondary IP addresses to assign to the pods, which is a common challenge for EKS IPv4 clusters.

Additionally, some workloads require to increase pod density 
in order to improve resources utilization. In Amazon EKS this is implemented by enabling VPC CNI prefix mode 

. To implement the prefix mode, instead of a single IP address, VPC CNI configures EC2 to assign /28 IP prefixes (16 IP addresses) to the ENI IP slots. When EC2 allocates a /28 IPv4 prefix to an ENI, it has to be a contiguous block of IP addresses from your subnet. If the subnet is fragmented due to an increased usage of the subnet by AWS services and worker nodes themselves, the prefix attachment may fail, essentially reducing the IP address space utilization.
Infrastructure and application traffic separation

Creating distinct network paths for different types of communication within a Kubernetes cluster is particularly valuable for organizations that need to maintain clear boundaries between their infrastructure management communications and their application workloads. Node-to-node communication typically includes cluster management traffic, infrastructure monitoring, while pod-to-pod communication handles application-specific data flows and service interactions.
Different network configuration for node and pod subnets

Applying a different network configuration to nodes and pods is also a common requirement for more complex systems. This includes customizing network address translation (SNAT) policies, placement on worker nodes and pods in public or private subnets, and defining different tagging to satisfy tools requirements.
Security considerations

The last two use cases are especially relevant use cases where traffic and access control are crucial to the security of the system.
Addressing the use cases

EKS Auto Mode provides advanced networking capabilities 
that allow us to implement granular network controls and traffic separation as well as multiple layers of network security utilizing the standard VPC features 
and Kubernetes-native network policies 

.

In this lab, we will show how to simplify solutions that address IP exhaustion and pod network isolation using EKS Auto Mode advanced networking capabilities. We will do so by:

    Adding a secondary CIDR block to the cluster VPC
    Creating new subnets from the new CIDR block
    Targeting the new subnets via EKS AutoMode NodePool and NodeClass configuration
    Configuring the application components to utilize the above configuration

Handle IP exhaustion
Review the EKS Auto Mode configuration

➤ Review the current nodes and pods IPs:

kubectl get nodes -o custom-columns=NAME:.metadata.name,INTERNAL-IP:.status.addresses[0].address
kubectl get pods -o wide

As expected, all pods and nodes belong to the same VPC CIDR - 192.168.0.0/16 we defined for our cluster during its creation:

NAME                  INTERNAL-IP
i-00e062e9543d100fc   192.168.9.85
i-016290d3063f80b19   192.168.123.101
i-0486577e8c56711ba   192.168.125.116
i-04ad8119e5de4ad75   192.168.17.136
i-078b82d5fe991368d   192.168.80.147
i-0b23214bab198a3f1   192.168.41.134
i-0e8c82c5d21eafcec   192.168.44.214
NAME                                         READY   STATUS    RESTARTS   AGE    IP               NODE
retail-store-app-carts-849f69cc8d-jndp8      1/1     Running   0          165m   192.168.50.130   i-0b23214bab198a3f1
retail-store-app-catalog-994d4889c-5fhrs     1/1     Running   0          58m    192.168.122.16   i-016290d3063f80b19
retail-store-app-catalog-994d4889c-j949t     1/1     Running   0          58m    192.168.28.240   i-00e062e9543d100fc
retail-store-app-catalog-994d4889c-k2csz     1/1     Running   0          57m    192.168.100.80   i-0486577e8c56711ba
retail-store-app-catalog-994d4889c-kq65z     1/1     Running   0          57m    192.168.15.128   i-04ad8119e5de4ad75
retail-store-app-catalog-994d4889c-t2mqk     1/1     Running   0          58m    192.168.49.48    i-0e8c82c5d21eafcec
retail-store-app-checkout-6df8f44b97-sx7xx   1/1     Running   0          61m    192.168.50.128   i-0b23214bab198a3f1
retail-store-app-orders-5fd7c6cf7-2xzq6      1/1     Running   0          61m    192.168.50.129   i-0b23214bab198a3f1
retail-store-app-ui-7fbf6d97b9-npm8x         1/1     Running   0          55m    192.168.50.132   i-0b23214bab198a3f1

Add a secondary CIDR to the cluster VPC

➤ Execute the following command to store the VPC ID in the terminal:

export VPC_ID=$(aws eks describe-cluster --name $DEMO_CLUSTER_NAME --query 'cluster.resourcesVpcConfig.vpcId' --output text)

➤ In the same terminal, execute the following command to explore all the CIDRs attached to the cluster VPC:

aws ec2 describe-vpcs \
  --vpc-ids $VPC_ID \
  --query 'Vpcs[0].CidrBlockAssociationSet[].CidrBlock'

Once again, as expected, there is only one, original, CIDR.

A reminder: EKS Auto Mode enables 

VPC CNI prefix delegation by default.

For the sake of the workshop, let's assume that we've exhausted enough IPs from the original CIDR block, so that it's impossible to provision new /28 blocks, which we require to reduce latency of pod provision or to increase pod density on our worker nodes.

The most straightforward way of dealing with the IP exhaustion issue is to attach a secondary CIDR to our VPC, create and tag subnets from that secondary CIDR and configure EKS to provision nodes and pods from these subnets.

Let's attach a secondary CIDR to the cluster VPC. Our options are outlined in this document 

. Since we've already used the entire 192.168.0.0/16 block, we need to select a different one.

➤ In the same terminal as the commands above (as we require the VPC_ID) execute the following command:

aws ec2 associate-vpc-cidr-block \
  --vpc-id ${VPC_ID} \
  --cidr-block 10.0.0.0/16

The above command is expected to fail with the following error:

An error occurred (InvalidVpc.Range) when calling the AssociateVpcCidrBlock operation: The CIDR '10.0.0.0/16' is restricted. Use a CIDR from the same private address range as the current VPC CIDR, or use a publicly-routable CIDR.
For additional restrictions, see https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html

This is because, as outlined in the VPC CIDR selection restrictions document 
, you can not combine different CIDRs from different RFC 1918 

blocks.

To resolve this and create more private IP address space for our pods we can use 

the 100.64.0.0/10 block instead.

➤ Execute the following command:

aws ec2 associate-vpc-cidr-block \
  --vpc-id ${VPC_ID} \
  --cidr-block 100.64.0.0/16

This should succeed now and produce the following output:

{
    "CidrBlockAssociation": {
        "AssociationId": "vpc-cidr-assoc-0df9d27b82606badc",
        "CidrBlock": "100.64.0.0/16",
        "CidrBlockState": {
            "State": "associating"
        }
    },
    "VpcId": "vpc-0413d277b700882f4"
}

➤ After a moment we can verify that the CIDR has been successfully attached to the cluster vpc by executing:

aws ec2 describe-vpcs \
  --vpc-ids $VPC_ID \
  --query 'Vpcs[0].CidrBlockAssociationSet[].CidrBlock'

The output should now contain both the original and the new CIDR:

[
    "192.168.0.0/16",
    "100.64.0.0/16"
]

Create additional subnets in the cluster VPC

We can now create 3 subnets in 3 different Availability Zones from the new secondary CIDR block, as recommended by the resilience best practices.

➤ Execute the following:

export VPC_ID=$(aws eks describe-cluster --name $DEMO_CLUSTER_NAME --query 'cluster.resourcesVpcConfig.vpcId' --output text)

export SUBNET_ID_A=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 100.64.0.0/19 \
  --availability-zone ${AWS_REGION}a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=eks-subnet-2a},{Key=advanced-networking,Value=1}]' \
  --query 'Subnet.SubnetId' \
  --output text)

export SUBNET_ID_B=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 100.64.32.0/19 \
  --availability-zone ${AWS_REGION}b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=eks-subnet-2b},{Key=advanced-networking,Value=1}]' \
  --query 'Subnet.SubnetId' \
  --output text)

export SUBNET_ID_C=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 100.64.64.0/19 \
  --availability-zone ${AWS_REGION}c \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=eks-subnet-2c},{Key=advanced-networking,Value=1}]' \
  --query 'Subnet.SubnetId' \
  --output text)

➤ Verify that the subnets have been created properly:

aws ec2 describe-subnets \
  --filters "Name=tag:advanced-networking,Values=1" \
  --query "Subnets[*].CidrBlock"

This should produce the following output:

[
    "100.64.64.0/19",
    "100.64.32.0/19",
    "100.64.0.0/19"
]

For pods to communicate with external resources, we need to associate our new subnets with a route table that defines the required configuration. In this case we can simply use the same route table we've used for the rest of the subnets.

➤ Execute the following (in that same terminal):

export ROUTE_TABLE_ID=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=${VPC_ID}" "Name=route.nat-gateway-id,Values=nat-*" \
  --query 'RouteTables[0].RouteTableId' \
  --output text)

aws ec2 associate-route-table \
  --route-table-id ${ROUTE_TABLE_ID} \
  --subnet-id ${SUBNET_ID_A}

aws ec2 associate-route-table \
  --route-table-id ${ROUTE_TABLE_ID} \
  --subnet-id ${SUBNET_ID_B}

aws ec2 associate-route-table \
  --route-table-id ${ROUTE_TABLE_ID} \
  --subnet-id ${SUBNET_ID_C}

Note that assigning a new VPC CIDR automatically updated the main route table to designate the 100.64.0.0/16 as a local target, in the same manner it does for the original 192.168.0.0/16 CIDR.

This expected output is as follows:

{
    "AssociationId": "rtbassoc-0bf6e8322df34a5a9",
    "AssociationState": {
        "State": "associated"
    }
}
{
    "AssociationId": "rtbassoc-0c742c5288fc4a532",
    "AssociationState": {
        "State": "associated"
    }
}
{
    "AssociationId": "rtbassoc-069e88b0bbe1d0fc0",
    "AssociationState": {
        "State": "associated"
    }
}

If we were to deploy an application, for its pods to be scheduled on instances provisioned in the new subnets, it would not actually work.

This is because Auto Mode autoscaling mechanism, via the built-in default NodeClass, doesn't "know" about them.
Create a custom NodeClass and NodePool to utilize the new subnets

To introduce the subnets and to make sure that network communication and connection to AWS services would work properly, we will target the corresponding tag (advanced-networking: '1', highlighted below), while re-using the IAM role and the original security group that allows the traffic between pods and the control plane.

➤ Create a new NodeClass that targets the new subnets:

1
2
3
4
5
6
7
8
9
10
11
12
13
14
cat << EOF > ~/environment/advanced-networking-nodeclass.yaml
apiVersion: eks.amazonaws.com/v1
kind: NodeClass
metadata:
  name: advanced-networking
spec:
  role: '${DEMO_CLUSTER_NODE_ROLE_NAME}'
  subnetSelectorTerms:
    - tags:
        advanced-networking: '1'
  securityGroupSelectorTerms:
    - tags:
        kubernetes.io/cluster/demo-cluster: owned
EOF

➤ Create a new NodePool that uses the NodeClass above:

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
cat << EOF > ~/environment/advanced-networking-nodepool.yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: advanced-networking
spec:
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 5m
  template:
    metadata:
      labels:
        role: advanced-networking
    spec:
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: advanced-networking
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: [amd64]
        - key: karpenter.sh/capacity-type
          operator: In
          values: [on-demand]
        - key: eks.amazonaws.com/instance-category
          operator: In
          values: [c, m, r]
        - key: eks.amazonaws.com/instance-cpu
          operator: In
          values: ['4', '8', '16', '32']
EOF

Note that we've added a custom label to the NodePool above to allow targeting these specific worker nodes with a nodeSelector in one of our application components. We only do the explicit targeting to demonstrate that these subnets are fully operational. In a real-world scenario new nodes and pods would consume IPs from the new subnets as required – most notably when there are no more IPs in the original subnets.

➤ Deploy the NodePool and the NodeClass:

kubectl apply -f ~/environment/advanced-networking-nodeclass.yaml
kubectl apply -f ~/environment/advanced-networking-nodepool.yaml

➤ Verify that the created components are ready to be used (the value in the READY column is True, which may take a couple of seconds):

kubectl get nodepool,nodeclass

To illustrate pods being provisioned in the new subnets, we'll re-deploy the UI application component.

➤ Execute the following command to create a custom values.yaml file:

1
2
3
4
cat << EOF > ~/environment/advanced-networking-values-ui.yaml
nodeSelector:
  role: advanced-networking
EOF

We added the node selector to illustrate topology spread across the new subnets to the file above and we will re-use the rest of the values from the previous Helm chart installation.

➤ Re-deploy the UI component:

1
2
3
4
5
helm upgrade retail-store-app-ui oci://public.ecr.aws/aws-containers/retail-store-sample-ui-chart \
  --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} \
  --values ~/environment/advanced-networking-values-ui.yaml \
  --reuse-values \
  --wait

Note that it will take a minute or so for the new instances to become operational.

➤ We can verify that both the UI pods and their nodes received IPs from the new subnets:

kubectl get nodes -o custom-columns=NAME:.metadata.name,IP:.status.addresses[0].address -l role=advanced-networking
kubectl get pods -l app.kubernetes.io/name=ui -o wide

This should provide an output similar to the following:

NAME                  IP
i-02b5a3d625f34288b   100.64.26.164
i-042c5454735eab393   100.64.58.201
i-0fcfb02997484b44c   100.64.84.98
NAME                                   READY   STATUS    RESTARTS   AGE   IP              NODE
retail-store-app-ui-69db5c4cdc-2tr6s   1/1     Running   0          20m   100.64.47.144   i-042c5454735eab393
retail-store-app-ui-69db5c4cdc-676hh   1/1     Running   0          25m   100.64.3.16     i-02b5a3d625f34288b
retail-store-app-ui-69db5c4cdc-wr8ll   1/1     Running   0          20m   100.64.79.48    i-0fcfb02997484b44c

We have now configured our subnets and the corresponding Auto Mode components to address the common IP exhaustion use case.

Sometimes, there may be additional considerations, such as traffic separation or application of different security controls between nodes and pods, that require us to take the network configuration and create a pod network isolation.
Isolate Pod Network

EKS Auto Mode allows to address these requirements by using subnet selection for pods 

NodeClass configuration.

➤ Update the advanced-networking NodeClass by executing:

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
cat << EOF > ~/environment/advanced-networking-nodeclass.yaml
apiVersion: eks.amazonaws.com/v1
kind: NodeClass
metadata:
  name: advanced-networking
spec:
  role: '${DEMO_CLUSTER_NODE_ROLE_NAME}'
  subnetSelectorTerms:
    - tags:
        kubernetes.io/role/internal-elb: '1'
  securityGroupSelectorTerms:
    - tags:
        kubernetes.io/cluster/demo-cluster: owned
  podSubnetSelectorTerms:
    - tags:
        advanced-networking: '1'
  podSecurityGroupSelectorTerms:
    - tags:
        kubernetes.io/cluster/demo-cluster: owned
EOF

The code above (see the highlighted lines) ensures that nodes and pods are placed into different subnets, while still allowing control plane and node-to-pod communications. We achieved that by configuring:

    the node-level subnet selector to target the original subnets
    the pod-level subnet selector (podSubnetSelectorTerms) to target the new subnets we created earlier
    the security group selector for both nodes and pods (identical in this example) to target a shared security group that allows traffic between the control plane, nodes and (now) pods

Note that for pods we've specifically targeted the private subnets, using kubernetes.io/role/elb: 1 tag as outlined in the documentation 

.

Note that using podSecurityGroupSelectorTerms is mandatory when using podSubnetSelectorTerms configuration. Alternatively, we could have used a different security group to also address the traffic separation use case in the same configuration.

Also note that EKS Auto Mode doesn't support 

Security Groups per Pod (SGPP).

Finally, keep in mind the following considerations for subnet selectors for pods:

    Reduced pod density: fewer pods can run on each node, because the IP slots on the node's primary EMI can no longer be used for pods
    Routing configuration: route table and network Access Control List (ACL) of the pod subnets are properly configured to allow the required communications

➤ Deploy the NodeClass (the advanced-networking NodePool doesn't require any changes):

kubectl apply -f ~/environment/advanced-networking-nodeclass.yaml

➤ Verify that the created components are ready to be used (the value in the READY column is True, which may take a couple of seconds):

kubectl get nodepool,nodeclass

Once applied, we don't actually need to do anything else, as Auto Mode will detect the drift 

(difference between the cluster state and the configuration outlined in the NodeClass) and reconcile the cluster to the desired state – node and pod subnet separation as we defined.

➤ Verify that the UI pods and their nodes received IPs from different subnets:

kubectl get nodes -o custom-columns=NAME:.metadata.name,IP:.status.addresses[0].address -l role=advanced-networking
kubectl get pods -l app.kubernetes.io/name=ui -o wide

Note that it will take a couple of minutes for the new non-drifted images to become operational.

This should provide an output similar to the following, showing nodes and pods IPs indeed belong to different subnets:

NAME                  IP
i-03fed7a34057e58c2   192.168.164.36
i-042f5d093908ac461   192.168.105.230
i-0a126a6040e63ec87   192.168.154.221
NAME                                   READY   STATUS    RESTARTS   AGE     IP              NODE
retail-store-app-ui-69db5c4cdc-knhv8   1/1     Running   0          5m22s   100.64.34.96    i-0a126a6040e63ec87
retail-store-app-ui-69db5c4cdc-nvtnh   1/1     Running   0          7m11s   100.64.4.96     i-042f5d093908ac461
retail-store-app-ui-69db5c4cdc-vgqm5   1/1     Running   0          6m21s   100.64.81.145   i-03fed7a34057e58c2

➤ We can verify that the application continues to function properly (showing the network setup is working) by extracting the ALB DNS and Ctrl/Cmd-clicking the printed URL:

export SHARED_ALB_URL=$(kubectl get ingress retail-store-shared-group-ui -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "The shared ALB is available at: http://$SHARED_ALB_URL"

Summary

In this lab, we've learned how to address common IP exhaustion and network separation use cases by performing the following:

    extending the cluster VPC with secondary CIDR blocks to provide additional IP address space
    creating new subnets in the secondary CIDR block
    associating a route table to ensure subnet-to-subnet traffic
    configuring EKS AutoMode NodePool and NodeClass resources to implement advanced networking (with or without podSubnetSelectorTerms and podSecurityGroupSelectorTerms)
