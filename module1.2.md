### Deploy a sample application

Overview | Review the cluster | Deploy the sample application | Summary
Overview

This section will help us understand the fundamentals of EKS Auto Mode and explain how to deploy applications. The power of Auto Mode is that we don't need to do any additional cluster configuration before launching our workloads.

EKS Auto Mode extends AWS management of Kubernetes components beyond the cluster itself, allowing AWS to set up and manage the infrastructure required for smooth operation of our workloads. We can delegate key infrastructure decisions and leverage the expertise of AWS for day-to-day operations. EKS Auto Mode includes many Kubernetes capabilities as managed components for compute autoscaling, Pod and service networking, application load balancing, cluster DNS, block storage, and GPU support.
Sample application

To demonstrate the capabilities EKS Auto Mode provides, we will use a sample web store application where customers can browse a catalog, add items to their cart, and complete an order through the checkout process. The application has several components, such as UI, catalog, orders, carts, and checkout services, along with backend databases that require persistent block storage, modeled as Kubernetes Deployments and StatefulSets. For its simplest deployment, the retail store app supports in-memory storage for persistence, which results in pods only for the services without the need to deploy pods that provide database or caching services.

To learn more about the application's design, refer to retail-store-app 

. We will use Kubernetes ingress to access UI applications outside of the cluster and will configure catalog applications to use Amazon EBS persistent storage. To demonstrate how Amazon EKS Auto Mode improves performance, provides scalability, and enhances availability, we will configure the UI application to support autoscaling, Pod topology spread constraints, and Pod disruption budgets (PDBs).

Retail web store application
Review the cluster

➤ Before deploying the application, check the cluster state again like we did in the Getting started section.

kubectl get pods --all-namespaces
kubectl get nodes

The expected result for both of the commands above is:

No resources found

➤ Verify that we do have access to the demo cluster by listing the CRDs (Custom Resource Definitions) in the cluster, this time listing all the available CRDs:

kubectl get crds

This should produce an output similar to the following:

NAME                                         CREATED AT
cninodes.eks.amazonaws.com                   2024-12-04T13:03:50Z
cninodes.vpcresources.k8s.aws                2024-12-04T13:00:46Z
ingressclassparams.eks.amazonaws.com         2024-12-04T13:03:49Z
nodeclaims.karpenter.sh                      2024-12-04T13:03:39Z
nodeclasses.eks.amazonaws.com                2024-12-04T13:03:39Z
nodediagnostics.eks.amazonaws.com            2024-12-04T13:03:39Z
nodepools.karpenter.sh                       2024-12-04T13:03:39Z
policyendpoints.networking.k8s.aws           2024-12-04T13:00:46Z
securitygrouppolicies.vpcresources.k8s.aws   2024-12-04T13:00:46Z
targetgroupbindings.eks.amazonaws.com        2024-12-04T13:03:49Z

As we can see, the cluster doesn't have any nodes or pods running on it. However, it has more CRDs than it had in the beginning of the workshop when we tested the cluster connectivity. This is the first change that happened after enabling EKS Auto Mode.

These CRDs enable several crucial capabilities of EKS Auto Mode:

    nodepools support provisioning compute
    ingressclassparams and targetgroupbinding allow exposing applications, and
    nodediagnostics provide diagnostics capabilities

To explore these capabilities, we will initially deploy the application in a manner that is self-contained in the Amazon EKS cluster, without using any AWS services that provision load balancers or managed databases. Over the course of the sections in this module, we will leverage different features of Amazon EKS Auto Mode to take advantage of broader AWS services and features to enable our retail store operations and functionality.
Deploy the sample application

We will use Helm 

charts for each of the components (catalog, order, UI etc.) to install the application. The code below contains a helm install command for each component and creates a values-ui.yaml file to specify UI application specific requirements and configure the components' endpoints.
Sample App Helm Values

➤ Let's execute the following commands to deploy the sample application:

cat << EOF > ~/environment/values-ui.yaml

app:
  theme: default
  endpoints:
    catalog: http://retail-store-app-catalog:80
    carts: http://retail-store-app-carts:80
    checkout: http://retail-store-app-checkout:80
    orders: http://retail-store-app-orders:80
EOF

helm upgrade -i retail-store-app-catalog oci://public.ecr.aws/aws-containers/retail-store-sample-catalog-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} --hide-notes
helm upgrade -i retail-store-app-orders oci://public.ecr.aws/aws-containers/retail-store-sample-orders-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} --hide-notes
helm upgrade -i retail-store-app-carts oci://public.ecr.aws/aws-containers/retail-store-sample-cart-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} --hide-notes
helm upgrade -i retail-store-app-checkout oci://public.ecr.aws/aws-containers/retail-store-sample-checkout-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} --hide-notes
helm upgrade -i retail-store-app-ui oci://public.ecr.aws/aws-containers/retail-store-sample-ui-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} -f ~/environment/values-ui.yaml --hide-notes

The commands above should produce helm outputs for each component of the app (catalog, orders, cart, checkout, and ui) similar to the following output:

[...]
NAME: retail-store-app-ui
LAST DEPLOYED: Mon Jan 20 17:42:52 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
[...]

Amazon EKS Auto Mode will evaluate the resource requirements of these Pods and determine the optimum compute to launch for our applications to run, considering any scheduling constraints configured.

➤ Wait for the nodes to become Ready by executing, in a separate VS Code terminal (it takes a couple of moments for the resources to provision, so wait 4 - 5 seconds and repeat if failed):

kubectl wait --for=condition=Ready nodes --all

After 10 - 15 seconds, the nodes should be ready:

node/i-0832abcbbc0911bee condition met

Note that EC2 instances created by EKS Auto Mode are different from other EC2 instances, as they are managed instances. These managed instances are owned by EKS and are more restricted. We can't directly access or install software on instances managed by EKS Auto Mode.

➤ Let's open a separate terminal by clicking on the + icon on the top right of the IDE, and watch for the application to become available:

kubectl wait --for=condition=available deployments --all

This is the expected result for an available application:

deployment.apps/retail-store-app-carts condition met
deployment.apps/retail-store-app-catalog condition met
deployment.apps/retail-store-app-checkout condition met
deployment.apps/retail-store-app-orders condition met
deployment.apps/retail-store-app-ui condition met

➤ Let's verify that all the components of the retail store applications are now running:

kubectl get pods

This should produce an output similar to this:

NAME                                        READY   STATUS    RESTARTS   AGE
retail-store-app-carts-5f5b7449f-tgscs      1/1     Running   0          91s
retail-store-app-catalog-dcb5d8d4c-5ftp9    1/1     Running   0          94s
retail-store-app-checkout-f5bb5c5bb-pkgg7   1/1     Running   0          90s
retail-store-app-orders-5fbb6b8576-x6vs9    1/1     Running   0          93s
retail-store-app-ui-7b7c8f6b94-2ltrs        1/1     Running   0          88s

We've also created a Service for each of our application components, allowing communication between the components.

➤ Let's execute the following command:

kubectl get svc

NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes                  ClusterIP   10.100.0.1       <none>        443/TCP   68m
retail-store-app-carts      ClusterIP   10.100.117.111   <none>        80/TCP    4m24s
retail-store-app-catalog    ClusterIP   10.100.192.87    <none>        80/TCP    4m28s
retail-store-app-checkout   ClusterIP   10.100.213.175   <none>        80/TCP    4m23s
retail-store-app-orders     ClusterIP   10.100.28.109    <none>        80/TCP    4m26s
retail-store-app-ui         ClusterIP   10.100.87.71     <none>        80/TCP    4m21s

These Services are internal to the cluster, so we cannot access them from the Internet or even the VPC. However, we can use port-forwarding 

to access an existing Pod in the EKS cluster to check that the application UI is working.

➤ Let's create a port-forwarding tunnel:

kubectl port-forward $(kubectl get pods \
 --selector=app.kubernetes.io/name=ui -o jsonpath='{.items[0].metadata.name}') 8080:8080

This should result in the following output:

Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
Handling connection for 8080
Handling connection for 8080

We can now access the application using the localhost endpoint http://localhost:8080. To do that, we can use the pop-up window that will appear on the bottom right side of the terminal. This will allow us to open the application in a new browser tab. The pop-up should look like the below:

port-forward-pop-up

The app might not render properly as it currently doesn't support proxy forwarding (which is how the VS Code IDE serves the port-forward command). We can ignore this as it has nothing to do with the cluster configuration, and the UI will be presented properly when we expose the app via load balancer (ALB) in a later module.

If the pop-up window doesn't appear (or you have accidentally closed it), you can open the application as follows:

    in an IDE terminal (other than the one that set up the port-forwarding) execute the following command echo "http://localhost:8080"

    Ctrl/Cmd-click the printed URL in the console

➤ We can close the port-forwarding connection by sending Ctrl/Cmd-C to the corresponding terminal.
Summary

This module demonstrated deploying a multi-component retail store application, showcasing how EKS Auto Mode automatically evaluates resource requirements and provisions appropriate compute resources.

In the next module, we will focus on EKS Auto Mode capabilities and their usage using our example application.
