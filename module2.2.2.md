On-Demand & Spot

Overview | Create NodePools | Deploy the application | Summary

In this section, we will explore the basics of EKS Auto Mode and dive deep into customization options.
Overview

Currently, all our compute nodes are running on On-Demand capacity. However, AWS EC2 offers multiple purchase options 

for running EKS workloads.

Amazon EC2 Spot Instances 

enable you to leverage unused EC2 capacity in the AWS cloud at discounts of up to 90% compared to On-Demand prices. Spot Instances are ideal for stateless, fault-tolerant, or flexible applications including big data workloads, containerized applications, CI/CD pipelines, web servers, high-performance computing (HPC), and test & development environments. These instances are particularly cost-effective when you have flexibility in application timing and can handle potential interruptions.

In this section, we will see how we can run workloads using both On-Demand and EC2 Spot Instances with a desired ratio to guarantee the base availability of On-Demand nodes, while leveraging Spot Instances for optimizing costs.
Create NodePools

We will create two NodePools that utilize Karpenter's capability to distribute workloads between on-demand and spot instances according to a defined ratio.

➤ Let's create the NodePools:

cat << EOF >~/environment/nodepool-ondemandspotsplit.yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: ondemand
  labels:
    app.kubernetes.io/managed-by: app-team
spec:
  disruption:
    consolidateAfter: 30s
    consolidationPolicy: WhenEmptyOrUnderutilized
  template:
    metadata:
      labels:
        EKSAutoNodePool: OnDemandSpotSplit
    spec:
      expireAfter: 336h
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: default
      requirements:
      - key: eks.amazonaws.com/instance-category
        operator: In
        values:
        - c
        - m
        - r
      - key: eks.amazonaws.com/instance-generation
        operator: Gt
        values:
        - "4"
      - key: kubernetes.io/arch
        operator: In
        values:
        - amd64
      - key: karpenter.sh/capacity-type
        operator: In
        values:
        - on-demand
      - key: capacity-spread
        operator: In
        values:
        - "1"
      taints:
      - effect: NoSchedule
        key: OnDemandSpotSplit
      terminationGracePeriod: 24h0m0s
---
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: spot
  labels:
    app.kubernetes.io/managed-by: app-team
spec:
  disruption:
    consolidateAfter: 30s
    consolidationPolicy: WhenEmptyOrUnderutilized
  template:
    metadata:
      labels:
        EKSAutoNodePool: OnDemandSpotSplit
    spec:
      expireAfter: 336h
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: default
      requirements:
      - key: eks.amazonaws.com/instance-category
        operator: In
        values:
        - c
        - m
        - r
      - key: eks.amazonaws.com/instance-generation
        operator: Gt
        values:
        - "4"
      - key: kubernetes.io/arch
        operator: In
        values:
        - amd64
      - key: karpenter.sh/capacity-type
        operator: In
        values:
        - spot
      - key: capacity-spread
        operator: In
        values:
        - "2"
        - "3"
        - "4"
        - "5"
      taints:
      - effect: NoSchedule
        key: OnDemandSpotSplit
      terminationGracePeriod: 24h0m0s
EOF

kubectl apply -f ~/environment/nodepool-ondemandspotsplit.yaml

The output should be similar:

nodepool.karpenter.sh/ondemand created
nodepool.karpenter.sh/spot created

We leverage Karpenter's node labeling and topology spread capabilities to implement a straightforward method for distributing workloads between on-demand and spot instances at a desired ratio.

To achieve this, we've created separate NodePools for Spot and On-Demand capacity types, each using distinct values for a custom label called capacity-spread. In our configuration, the spot NodePool has four unique values while the On-Demand NodePool has one value. When workloads are spread evenly across this label, we achieve a 4:1 ratio of spot to On-Demand nodes.
Deploy the application

Now, we'll configure the catalog component to distribute its replicas across Spot Instances. We'll set up 5 replicas of our catalog application and use the capacity-spread label to achieve our desired 4:1 ratio of spot to On-Demand nodes.
Configure and re-deploy the catalog component

➤ Let's configure and re-deploy the component:

cat << EOF >~/environment/values-catalog.yaml
replicaCount: 5
  
topologySpreadConstraints:
  - maxSkew: 1
    minDomains: 5
    topologyKey: capacity-spread
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: catalog

nodeSelector:
  EKSAutoNodePool: OnDemandSpotSplit
tolerations:
- key: "OnDemandSpotSplit"
  operator: "Exists"
EOF

helm upgrade -f ~/environment/values-catalog.yaml retail-store-app-catalog oci://public.ecr.aws/aws-containers/retail-store-sample-catalog-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} --hide-notes

The output should be similar to:

Pulled: public.ecr.aws/aws-containers/retail-store-sample-catalog-chart:1.1.0
Digest: sha256:0dc16ac63a8f32f309ea7d69f58973520c156305f69f2703fcb5564da6b67eb6
Release "retail-store-app-catalog" has been upgraded. Happy Helming!
NAME: retail-store-app-catalog
LAST DEPLOYED: Sun Jan 19 23:42:07 2025
NAMESPACE: default
STATUS: deployed
REVISION: 2

Let's verify that the catalog component pods are running on Spot instances.

➤ Execute these commands to check the nodes and pods:

kubectl get node -L karpenter.sh/capacity-type --no-headers | while read node status roles age version capacity_type; do
echo "Pods on node $node (Capacity Type: $capacity_type):"
  kubectl get pods --all-namespaces --field-selector spec.nodeName=$node -l app.kubernetes.io/instance=retail-store-app-catalog
echo "-----------------------------------"
done

Note that it may take a minute for the instances to be created and become operational and that nodes that will be removed after consolidation will appear empty.

Pods on node i-02e25edfdcf052a8d (Capacity Type: on-demand):
No resources found
-----------------------------------
Pods on node i-059c896e2ddc3787f (Capacity Type: spot):
NAMESPACE   NAME                                        READY   STATUS    RESTARTS   AGE
default     retail-store-app-catalog-7568d4cffb-svztn   1/1     Running   0          2m13s
-----------------------------------
Pods on node i-074da6ae5127bf5dd (Capacity Type: on-demand):
No resources found
-----------------------------------
Pods on node i-0865727e5109e8eda (Capacity Type: on-demand):
NAMESPACE   NAME                                        READY   STATUS    RESTARTS   AGE
default     retail-store-app-catalog-7568d4cffb-ft88b   1/1     Running   0          2m16s
-----------------------------------
Pods on node i-095f16c14ff78d4e0 (Capacity Type: spot):
NAMESPACE   NAME                                        READY   STATUS    RESTARTS   AGE
default     retail-store-app-catalog-7568d4cffb-gkvqj   1/1     Running   0          2m59s
-----------------------------------
Pods on node i-0995ee2810f2eda4b (Capacity Type: on-demand):
No resources found
-----------------------------------
Pods on node i-0b36d75cd120def01 (Capacity Type: spot):
NAMESPACE   NAME                                        READY   STATUS    RESTARTS   AGE
default     retail-store-app-catalog-7568d4cffb-s6t4f   1/1     Running   0          3m2s
-----------------------------------
Pods on node i-0d7d6046307eecb7e (Capacity Type: spot):
NAMESPACE   NAME                                        READY   STATUS    RESTARTS   AGE
default     retail-store-app-catalog-7568d4cffb-4n4xs   1/1     Running   0          3m3s

We can confirm that the catalog app pods are successfully distributed across both Spot and On-demand capacity types.
Summary

In this lab, we've explored how to effectively combine On-Demand and Spot instances in EKS Auto Mode. We implemented a split-ratio strategy using two NodePools and the capacity-spread label, configuring the Spot NodePool with four unique spread values and the On-Demand NodePool with one value to achieve a 4:1 ratio.

To demonstrate this configuration, we deployed our catalog application with 5 replicas and used topology spread constraints to distribute the pods according to our defined ratio. This approach demonstrates how to balance reliability with cost optimization in EKS cluster management by maintaining a baseline of stable On-Demand capacity while leveraging cost-effective Spot instances for workloads that can handle interruptions.

This lab illustrated how to achieve an optimal balance between reliability and cost efficiency in your EKS cluster management strategy.
