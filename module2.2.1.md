Graviton

Overview | Create a Graviton NodePool | Summary

This section will guide you through the basics of EKS Auto Mode compute customization options and explain in detail customization using Graviton.
Overview
Graviton instances

AWS Graviton-based processors 

deliver the best price-performance for cloud workloads and offer up to a 40% better price-performance over comparable x86-based Amazon EC2 instances. Graviton processors also use up to 60% less energy than comparable EC2 instances for the same performance. AWS Graviton-based Amazon EC2 instances provide the best price-performance for a wide variety of Linux-based workloads, such as application servers, microservices, high-performance computing (HPC), CPU-based machine learning inference, video encoding, electronic design automation (EDA), gaming, open-source databases, in-memory caches, etc.
Review the current NodePool

In this section, we will create a new, custom general purpose NodePool to provision AWS Graviton instances. Before we create the new NodePool, let's review the existing general-purpose NodePool and the current state of the nodes available in the cluster.

➤ Execute the following command:

kubectl get nodepool general-purpose -o yaml

The configuration of the general-purpose NodePool should look like this:


apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  annotations:
    karpenter.sh/nodepool-hash: "4012513481623584108"
    karpenter.sh/nodepool-hash-version: v3
  creationTimestamp: "2025-01-15T09:32:29Z"
  generation: 1
  labels:
    app.kubernetes.io/managed-by: eks
  name: general-purpose
  resourceVersion: "1097754"
  uid: 9b1c4ad0-d42d-4c63-bd96-b0a201aeec0e
spec:
  disruption:
    budgets:
    - nodes: 10%
    consolidateAfter: 30s
    consolidationPolicy: WhenEmptyOrUnderutilized
  template:
    metadata: {}
    spec:
      expireAfter: 336h
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: default
      requirements:
      - key: karpenter.sh/capacity-type
        operator: In
        values:
        - on-demand
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
      - key: kubernetes.io/os
        operator: In
        values:
        - linux
      terminationGracePeriod: 24h0m0s
status:
  conditions:
  - lastTransitionTime: "2025-01-15T09:32:43Z"
    message: ""
    observedGeneration: 1
    reason: ValidationSucceeded
    status: "True"
    type: ValidationSucceeded
  - lastTransitionTime: "2025-01-15T09:32:44Z"
    message: ""
    observedGeneration: 1
    reason: NodeClassReady
    status: "True"
    type: NodeClassReady
  - lastTransitionTime: "2025-01-15T09:32:44Z"
    message: ""
    observedGeneration: 1
    reason: Ready
    status: "True"
    type: Ready
  resources:
    cpu: "4"
    ephemeral-storage: 163708Mi
    hugepages-1Gi: "0"
    hugepages-2Mi: "0"
    memory: 7717496Ki
    nodes: "2"
    pods: "54"

➤ View the current instances processor architecture:

kubectl get nodes -L kubernetes.io/arch

with an output similar to this:

NAME                  STATUS   ROLES    AGE   VERSION               ARCH
...
i-07c121b110f507617   Ready    <none>   15h   v1.32.3-eks-7636447   amd64
i-0c90d33aa6fccecff   Ready    <none>   16h   v1.32.3-eks-7636447   amd64

As we can see, all current nodes are using EC2 instances with the amd64 processor architecture, as specified by the general-purpose NodePool.

Your output may vary slightly since EKS Auto Mode provisions instances according to the requirements defined in the node pool.
Create a Graviton NodePool

Now we'll create a new NodePool that includes arm64 (Graviton) architecture in the kubernetes.io/arch requirement.

This configuration enables Auto Mode managed Karpenter to evaluate each new Pod's nodeAffinity or nodeSelector for CPU architecture requirements. If needed, Karpenter will launch a new Graviton node for pending pods. We'll also add a taint with key:GravitonOnly and effect:NoSchedule to our Graviton NodePool to ensure only pods with matching tolerations are scheduled on these nodes.

➤ Create the new NodePool definition:

cat << EOF >~/environment/nodepool-graviton.yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: graviton
  labels:
    app.kubernetes.io/managed-by: app-team
spec:
  disruption:
    budgets:
    - nodes: 10%
    consolidateAfter: 30s
    consolidationPolicy: WhenEmptyOrUnderutilized
  template:
    metadata: {}
    spec:
      expireAfter: 336h
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: default
      requirements:
      - key: karpenter.sh/capacity-type
        operator: In
        values:
        - on-demand
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
        - arm64
      taints:
      - effect: NoSchedule
        key: GravitonOnly
      terminationGracePeriod: 24h0m0s
  limits:
    cpu: "1000"
    memory: 1000Gi
EOF

kubectl apply -f ~/environment/nodepool-graviton.yaml

This should result in this output:

nodepool.karpenter.sh/graviton created

Run pods on Graviton

With our Graviton NodePool in place, let's configure our application's UI component to utilize it.

➤ First, let's examine the current configuration of the UI component pods:

kubectl describe pod --selector app.kubernetes.io/name=ui

This should produce an output similar to the following:

Name:             retail-store-app-ui-697bbcdb5-jf6bs
Namespace:        default
Priority:         0
Service Account:  retail-store-app-ui
Node:             i-07c121b110f507617/20.0.144.196
Start Time:       Fri, 17 Jan 2025 15:17:29 +0000
Labels:           app.kuberneres.io/owner=retail-store-sample
                  app.kubernetes.io/component=service
                  app.kubernetes.io/instance=retail-store-app
                  app.kubernetes.io/name=ui
                  pod-template-hash=697bbcdb5
Annotations:      prometheus.io/path: /actuator/prometheus
                  prometheus.io/port: 8080
                  prometheus.io/scrape: true
Status:           Running
[...]
Node-Selectors:               <none>
Tolerations:                  node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                              node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Topology Spread Constraints:  kubernetes.io/hostname:ScheduleAnyway when max skew 1 is exceeded for selector app.kubernetes.io/name=ui
                              topology.kubernetes.io/zone:ScheduleAnyway when max skew 1 is exceeded for selector app.kubernetes.io/name=ui
Events:                       <none>

The Pod is Running and has no custom tolerations configured.

Kubernetes automatically adds tolerations for node.kubernetes.io/not-ready and node.kubernetes.io/unreachable with tolerationSeconds=300 unless explicitly set. These tolerations allow Pods to remain bound to nodes for 5 minutes after detecting these issues.

Let's update our UI component to bind its pods to our Graviton NodePool.

We've tainted the NodePool with key:GravitonOnly and it automatically adds a karpenter.sh/nodepool label.

The following values-ui.yaml contains the changes needed to our UI app configuration in order to enable this setup.
Re-deploy the UI component

cat << EOF >~/environment/values-ui.yaml
app:
  theme: default
  endpoints:
    catalog: http://retail-store-app-catalog:80
    carts: http://retail-store-app-carts:80
    checkout: http://retail-store-app-checkout:80
    orders: http://retail-store-app-orders:80

topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: ui

autoscaling:
  enabled: false
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

nodeSelector:
  karpenter.sh/nodepool: graviton
tolerations:
- key: "GravitonOnly"
  operator: "Exists"
EOF

helm upgrade -f ~/environment/values-ui.yaml retail-store-app-ui oci://public.ecr.aws/aws-containers/retail-store-sample-ui-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} --hide-notes

Note that the UI component will revert to its original configuration in the values-ui.yaml file, which specified a single replica.

This should produce the following output:

Pulled: public.ecr.aws/aws-containers/retail-store-sample-ui-chart:1.1.0
Digest: sha256:5cd721c10214c306b06c7223367f626f21a8d471eee8f0a576742426f84141f2
Release "retail-store-app-ui" has been upgraded. Happy Helming!
NAME: retail-store-app-ui
LAST DEPLOYED: Sat Jan 18 23:54:22 2025
NAMESPACE: default
STATUS: deployed
REVISION: 4

➤ Before examining the new Graviton nodes, watch for all UI component pods to become ready:

kubectl wait --for=condition=Ready pod -l app.kubernetes.io/instance=retail-store-app-ui --namespace default --timeout=300s
kubectl get pods -l app.kubernetes.io/instance=retail-store-app-ui

➤ Now check the status of our EKS cluster nodes and UI component pods:

kubectl get nodes -L kubernetes.io/arch -L karpenter.sh/nodepool
kubectl get pods -l app.kubernetes.io/name=ui -o wide

With the expected output containing arm64 instances, similar to below:

NAME                  STATUS   ROLES    AGE    VERSION               ARCH    NODEPOOL
i-078b82d5fe991368d   Ready    <none>   45s    v1.32.5-eks-98436be   arm64   system
i-0b23214bab198a3f1   Ready    <none>   105m   v1.32.5-eks-98436be   amd64   general-purpose
i-0f67fef87c747070d   Ready    <none>   104s   v1.32.5-eks-98436be   arm64   graviton
NAME                                   READY   STATUS    RESTARTS   AGE    IP                NODE
retail-store-app-ui-86df66db68-744sn   1/1     Running   0          2m7s   192.168.190.160   i-0f67fef87c747070d

As you can see, the UI component pods are now running on the Graviton NodePool. You can also see the taint on the node using the kubectl describe node command and the matching tolerations on the pods using the kubectl describe pod command.

Note that it may take a couple of minutes for the cluster to arrive to a state similar to the above.
Summary

In this lab, we explored using AWS Graviton instances in EKS Auto Mode for improved performance and cost efficiency.

We created a dedicated Graviton NodePool configured for arm64 architecture instances with a GravitonOnly taint to control Pod scheduling. We then modified our application by updating the UI component's configuration with the necessary node selector and toleration to enable running on Graviton instances.

In the next lab we will explore combining On-Demand instances with Spot Instances for additional cost optimization.
