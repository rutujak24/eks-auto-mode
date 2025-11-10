Exposing Applications

Overview | Expose the application | Summary
Overview

In this hands-on lab, we'll learn how to expose applications in Amazon EKS Auto Mode using AWS's managed load balancing solutions.

We will work through a practical scenario to:

    Configure an Application Load Balancer (ALB) using Kubernetes Ingress resources
    Set up a Network Load Balancer (NLB) using Kubernetes Service resources
    Implement path-based routing to share a single ALB across multiple services
    Observe load balancing in action with custom response headers showing which pods handle requests

Expose the application

EKS Auto Mode simplifies the process by automatically managing the lifecycle of the ALBs and NLBs that are required for our application. As EKS Auto Mode is Kubernetes conformant, it allows us to use the same Kubernetes constructs of Service 
and Ingress 

to provision those Load Balancers.

In the remainder of this module, we'll learn how to provision those load balancers with Auto Mode.
Step 1: Set Up IngressClass for ALB

Since Ingresses can be implemented by different controllers, each Ingress should specify a class, a reference to an IngressClass resource that contains additional configuration including the name of the controller that should implement the class.

IngressClass resources contain an optional parameters field. This can be used to reference additional implementation-specific configuration for this class.

To target the EKS Auto Mode ALB load balancing capability controller, we will create an IngressClassParams, which allows us to define an AWS specific configuration for our ALB such as certificates to use, the subnets to use for the ALB ENIs, or the ingress group 

configuration to group together multiple ingress objects into a single ALB.

The supported configurations for the IngressClassParams objects are listed 

in EKS Auto Mode documentation. Additionally, we will create IngressClass that will use the IngressClassParams and point to the EKS Auto Mode capability. This is a one-time setup required for using ALBs in our cluster. Notice the spec.controller definition in the IngressClass below.

➤ Create the IngressClass and IngressClassParams to further configure the EKS Auto Mode load balancing capability:

cat << EOF >~/environment/ingress.yaml
apiVersion: eks.amazonaws.com/v1
kind: IngressClassParams
metadata:
  name: eks-auto-alb
spec:
  scheme: internet-facing
---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: eks-auto-alb
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: eks.amazonaws.com/alb
  parameters:
    apiGroup: eks.amazonaws.com
    kind: IngressClassParams
    name: eks-auto-alb
EOF

kubectl apply -f ~/environment/ingress.yaml

This should produce the following output:

ingressclassparams.eks.amazonaws.com/eks-auto-alb created
ingressclass.networking.k8s.io/eks-auto-alb created

➤ Verify that the resources have been created:

kubectl get ingressclass,ingressclassparams

The output should look like the one below:

NAME                                          CONTROLLER              PARAMETERS                                          AGE
ingressclass.networking.k8s.io/eks-auto-alb   eks.amazonaws.com/alb   IngressClassParams.eks.amazonaws.com/eks-auto-alb   56s

NAME                                                GROUP-NAME   SCHEME            IP-ADDRESS-TYPE   AGE
ingressclassparams.eks.amazonaws.com/eks-auto-alb                internet-facing                     56s

Note the controller used, as well as the SCHEME defined (internet-facing in our case) which is based on our configuration above.
Step 2: Deploy the Retail Store Application

We will now update the UI component to provision an ALB by creating an Ingress object, as we can see with the ingress configuration in the component's Helm chart values.

➤ Re-deploy the UI component with custom values:

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
33
34
35
36
37
38
39
cat << EOF >~/environment/values-ui.yaml
app:
  theme: default
  endpoints:
    catalog: http://retail-store-app-catalog:80
    carts: http://retail-store-app-carts:80
    checkout: http://retail-store-app-checkout:80
    orders: http://retail-store-app-orders:80

replicaCount: 3

autoscaling:
  enabled: false
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

topologySpreadConstraints:
   - maxSkew: 1
     minDomains: 3
     topologyKey: topology.kubernetes.io/zone
     whenUnsatisfiable: DoNotSchedule
     labelSelector:
       matchLabels:
         app.kubernetes.io/name: ui

ingress:
  enabled: true
  className: eks-auto-alb
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: /actuator/health/liveness
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
    alb.ingress.kubernetes.io/success-codes: '200-399'
EOF

helm upgrade -f ~/environment/values-ui.yaml retail-store-app-ui oci://public.ecr.aws/aws-containers/retail-store-sample-ui-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} --hide-notes

This should produce an output similar to the following:

Pulled: public.ecr.aws/aws-containers/retail-store-sample-ui-chart:1.1.0
Digest: sha256:5cd721c10214c306b06c7223367f626f21a8d471eee8f0a576742426f84141f2
Release "retail-store-app-ui" has been upgraded. Happy Helming!
NAME: retail-store-app-ui
LAST DEPLOYED: Sun Feb 16 21:30:01 2025
NAMESPACE: default
STATUS: deployed
REVISION: 5

➤ Wait for all deployments to be ready by using the following command:

kubectl wait --for=condition=available deployments retail-store-app-ui --all

Example output:

deployment.apps/retail-store-app-ui condition met

Step 3: Access the UI application with the provisioned ALB

The Ingress object that we've deployed in the previous step gets translated by EKS Auto Mode into an ALB with the appropriate configurations we've defined in the Ingress object itself, and in the IngressClassParams above.
ALB provisioning takes couple of minutes

Note that initial ALB provision and targets registration may take several minutes, so the following command will wait for the completion and not exit until the load balancer is available.

➤ Retrieve the ALB's DNS endpoint by executing the following command:

export ALB_URL=$(kubectl get ingress retail-store-app-ui -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

aws elbv2 wait load-balancer-available --load-balancer-arns $(aws elbv2 describe-load-balancers --query 'LoadBalancers[?DNSName==`'"$ALB_URL"'`].LoadBalancerArn' --output text)

echo "Your application is available at: http://${ALB_URL}"

At this point, we can use the URL to access the application in a new browser window.
Step 4: Expose Catalog Service Using NLB

In the previous steps, we've used EKS Auto Mode to provision an ALB. We'll now experience how to use EKS Auto Mode to provision NLB using the Kubernetes Service object.

➤ Execute the following command.

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
kubectl apply -f - << EOF
apiVersion: v1
kind: Service
metadata:
  name: catalog-nlb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
  selector:
    app.kubernetes.io/name: catalog
EOF

➤ Wait for the NLB to be provisioned and get its URL:

export NLB_URL=$(kubectl get service catalog-nlb -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

aws elbv2 wait load-balancer-available --load-balancer-arns $(aws elbv2 describe-load-balancers --query 'LoadBalancers[?
DNSName==`'"$NLB_URL"'`].LoadBalancerArn' --output text)
echo "The catalog service is also available at: http://${NLB_URL}"

Note that we will test the catalog component API via the NLB in step 5.2.

While you wait for the NLB to come up, you can continue to the Step 5.1 to test application access through the already operational ALB.
Step 5.1: Test Application Load Balancer Access

Let's test access to our application and observe load balancing in action. The catalog service has been configured with 5 replicas in the compute module under "On-Demand & Spot Split Ratio". We will use this to demonstrate load distribution.

➤ First, verify that all catalog pods are running:

kubectl get pods -l app.kubernetes.io/name=catalog,app.kubernetes.io/component=service

You should see that all of the catalog's component pods are in Running state. Now, let's use two terminal windows to observe the load balancing behavior.

➤ In one terminal, watch the logs from all catalog pods:

kubectl logs -f -l app.kubernetes.io/name=catalog,app.kubernetes.io/component=service --prefix=true

➤ In another terminal, generate some traffic through the ALB:

# Get the ALB URL
export ALB_URL=$(kubectl get ingress retail-store-app-ui -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "The application is available at: http://${ALB_URL}"

# Generate traffic to see load balancing across pods
CURL_CMD=$(which curl)
for i in {1..15}; do
  echo "Sending request $i..."
  $CURL_CMD -s "http://${ALB_URL}/catalog?request=$i" > /dev/null
  sleep 2
done

You should see detailed logs in your first terminal showing requests being distributed across all three catalog pods. Each log line shows:

    Which Pod handled the request (in the prefix)
    HTTP method and path
    Response status code
    Request processing time
    Client IP address

Example log output showing distribution across pods:

[pod/retail-store-app-catalog-994d4889c-rkk9f/catalog]
[pod/retail-store-app-catalog-994d4889c-rkk9f/catalog] 2025/11/05 17:45:23 /appsrc/repository/repository.go:204
[pod/retail-store-app-catalog-994d4889c-rkk9f/catalog] [0.181ms] [rows:4] SELECT * FROM `tags` ORDER BY display_name asc
[pod/retail-store-app-catalog-994d4889c-rkk9f/catalog] [GIN] 2025/11/05 - 17:45:23 | 200 |     288.903µs |   192.168.95.71 | GET      "/catalog/tags"
[pod/retail-store-app-catalog-994d4889c-rkk9f/catalog]
[pod/retail-store-app-catalog-994d4889c-rkk9f/catalog] 2025/11/05 17:45:23 /appsrc/repository/repository.go:190
[pod/retail-store-app-catalog-994d4889c-rkk9f/catalog] [0.080ms] [rows:1] SELECT count(*) FROM `products`
[pod/retail-store-app-catalog-994d4889c-rkk9f/catalog] [GIN] 2025/11/05 - 17:45:23 | 200 |     157.362µs |   192.168.95.71 | GET      "/catalog/size?tags="
[pod/retail-store-app-catalog-994d4889c-rkk9f/catalog]
[pod/retail-store-app-catalog-994d4889c-rkk9f/catalog] 2025/11/05 17:45:23 /appsrc/repository/repository.go:154
[pod/retail-store-app-catalog-994d4889c-rkk9f/catalog] [0.110ms] [rows:6] SELECT * FROM `product_tags` WHERE `product_tags`.`product_id` IN ("a1258cd2-176c-4507-ade6-746dab5ad625","d4edfedb-dbe9-4dd9-aae8-009489394955","79bce3f3-935f-4912-8c62-0d2f3e059405","8757729a-c518-4356-8694-9e795a9b3237","4f18544b-70a5-4352-8e19-0d070f46745d","d77f9ae6-e9a8-4a3e-86bd-b72af75cbc49")
[pod/retail-store-app-catalog-994d4889c-rkk9f/catalog]
[pod/retail-store-app-catalog-994d4889c-rkk9f/catalog] 2025/11/05 17:45:23 /appsrc/repository/repository.go:154
[pod/retail-store-app-catalog-994d4889c-rkk9f/catalog] [0.082ms] [rows:4] SELECT * FROM `tags` WHERE `tags`.`name` IN ("clothing","food","vehicles","accessories")
[pod/retail-store-app-catalog-994d4889c-rkk9f/catalog]
[pod/retail-store-app-catalog-994d4889c-rkk9f/catalog] 2025/11/05 17:45:23 /appsrc/repository/repository.go:154
[pod/retail-store-app-catalog-994d4889c-rkk9f/catalog] [0.517ms] [rows:6] SELECT * FROM `products` ORDER BY products.name asc LIMIT 6
[pod/retail-store-app-catalog-994d4889c-rkk9f/catalog] [GIN] 2025/11/05 - 17:45:23 | 200 |     609.082µs |   192.168.95.71 | GET      "/catalog/products?order=&page=1&size=6&tags="
[pod/retail-store-app-catalog-994d4889c-d6ns4/catalog]
[pod/retail-store-app-catalog-994d4889c-d6ns4/catalog] 2025/11/05 17:45:25 /appsrc/repository/repository.go:204
[pod/retail-store-app-catalog-994d4889c-d6ns4/catalog] [0.206ms] [rows:4] SELECT * FROM `tags` ORDER BY display_name asc
[pod/retail-store-app-catalog-994d4889c-d6ns4/catalog] [GIN] 2025/11/05 - 17:45:25 | 200 |     313.707µs |    192.168.99.1 | GET      "/catalog/tags"
[pod/retail-store-app-catalog-994d4889c-d6ns4/catalog]
[pod/retail-store-app-catalog-994d4889c-d6ns4/catalog] 2025/11/05 17:45:25 /appsrc/repository/repository.go:190
[pod/retail-store-app-catalog-994d4889c-d6ns4/catalog] [0.058ms] [rows:1] SELECT count(*) FROM `products`
[pod/retail-store-app-catalog-994d4889c-d6ns4/catalog] [GIN] 2025/11/05 - 17:45:25 | 200 |     129.603µs |    192.168.99.1 | GET      "/catalog/size?tags="
[pod/retail-store-app-catalog-994d4889c-d6ns4/catalog]
[pod/retail-store-app-catalog-994d4889c-d6ns4/catalog] 2025/11/05 17:45:25 /appsrc/repository/repository.go:154
[pod/retail-store-app-catalog-994d4889c-d6ns4/catalog] [0.093ms] [rows:6] SELECT * FROM `product_tags` WHERE `product_tags`.`product_id` IN ("a1258cd2-176c-4507-ade6-746dab5ad625","d4edfedb-dbe9-4dd9-aae8-009489394955","79bce3f3-935f-4912-8c62-0d2f3e059405","8757729a-c518-4356-8694-9e795a9b3237","4f18544b-70a5-4352-8e19-0d070f46745d","d77f9ae6-e9a8-4a3e-86bd-b72af75cbc49")
[pod/retail-store-app-catalog-994d4889c-d6ns4/catalog]
[pod/retail-store-app-catalog-994d4889c-d6ns4/catalog] 2025/11/05 17:45:25 /appsrc/repository/repository.go:154
[pod/retail-store-app-catalog-994d4889c-d6ns4/catalog] [0.079ms] [rows:4] SELECT * FROM `tags` WHERE `tags`.`name` IN ("clothing","food","vehicles","accessories")
[pod/retail-store-app-catalog-994d4889c-d6ns4/catalog]
[pod/retail-store-app-catalog-994d4889c-d6ns4/catalog] 2025/11/05 17:45:25 /appsrc/repository/repository.go:154
[pod/retail-store-app-catalog-994d4889c-d6ns4/catalog] [0.461ms] [rows:6] SELECT * FROM `products` ORDER BY products.name asc LIMIT 6
[pod/retail-store-app-catalog-994d4889c-d6ns4/catalog] [GIN] 2025/11/05 - 17:45:25 | 200 |     528.831µs |    192.168.99.1 | GET      "/catalog/products?order=&page=1&size=6&tags="

The catalog service uses the Gin web framework which provides detailed request logging. We can observe:

    Requests being distributed across different Pods (see the Pod names in the log prefix)
    Multiple requests being made for each page load (tags, size, and catalog data)
    Response times for each request
    Client IPs making the requests

Step 5.2: Test Network Load Balancer Access

➤ To test the access to the catalog service directly through the NLB provisioned earlier, ensure that the kubectl logs command is still running on the other terminal:

kubectl logs -f -l app.kubernetes.io/name=catalog,app.kubernetes.io/component=service --prefix=true

➤ Now we can test from the access to the catalog service through the NLB by using the NLB DNS name with the appended URI below:

export NLB_URL=$(kubectl get service catalog-nlb -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://${NLB_URL}/catalog/products | jq

You should expect to see a JSON response from the catalog service, as well as logs from that request on the other terminal.
Step 6: Share ALB Across Multiple Services - Multiple Ingress Pattern

In some use-cases we need to ensure that multiple ingress objects don't create multiple ALBs but rather use the same ALB with multiple routing rules. This is where EKS Auto Mode supports ingress grouping using the IngressClassParams object (see reference in the documentation 

). In this step, we will create a new IngressClass object with new IngressClassParams that supports such grouping. For demonstration purposes, we will then create 2 ingress objects: one for the ui component and one for the catalog service. With this configuration, since we've configured the grouping on the IngressClassParams, a single ALB will be created pointing to both of those services. Follow the steps below to achieve that:

➤ 1. Create a new IngressClassParams and IngressClass with group.name configuration:

cat << EOF >~/environment/ingress-class-group.yaml
apiVersion: eks.amazonaws.com/v1
kind: IngressClassParams
metadata:
  name: eks-auto-alb-group-retail
spec:
  scheme: internet-facing
  group:
    name: retail
---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: eks-auto-alb-group-retail
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: eks.amazonaws.com/alb
  parameters:
    apiGroup: eks.amazonaws.com
    kind: IngressClassParams
    name: eks-auto-alb-group-retail
EOF

kubectl apply -f ~/environment/ingress-class-group.yaml

➤ 2. Create an ingress object for the ui component (note the use of the newly created IngressClass eks-auto-alb-group-retail ):

kubectl apply -f - << EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: retail-store-shared-group-ui
  annotations:
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /actuator/health/liveness
spec:
  ingressClassName: eks-auto-alb-group-retail
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: retail-store-app-ui
                port:
                  number: 80
EOF

➤ 3. Create a second ingress object for the catalog component:

kubectl apply -f - << EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: retail-store-shared-group-catalog
  annotations:
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /health
spec:
  ingressClassName: eks-auto-alb-group-retail
  rules:
  - http:
      paths:
      - path: /catalog
        pathType: Prefix
        backend:
          service:
            name: retail-store-app-catalog
            port:
              number: 80
EOF

➤ 4. Verify the ingresses had been created:

kubectl get ingress

Note that the output ADDRESS of both ingresses of the catalog and the ui has the same DNS. This is because of the IngressClassParams group configuration we've used above.

NAME                                 CLASS                       HOSTS   ADDRESS                                                                 PORTS   AGE
retail-store-app-ui                  eks-auto-alb                *       k8s-default-retailst-70051948b0-467757540.us-west-2.elb.amazonaws.com   80      16m
retail-store-backend-group-catalog   eks-auto-alb-group-retail   *       k8s-retail-5620a3cc93-1836759971.us-west-2.elb.amazonaws.com            80      30s
retail-store-backend-group-ui        eks-auto-alb-group-retail   *       k8s-retail-5620a3cc93-1836759971.us-west-2.elb.amazonaws.com            80      11m

➤ 5. Extract the ALB DNS (note that because both ingresses have the same DNS, we can randomly choose one of them):

# Get the shared ALB URL
export SHARED_ALB_URL=$(kubectl get ingress retail-store-shared-group-ui -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
# wait for the shared ALB to become active
aws elbv2 wait load-balancer-available --load-balancer-arns $(aws elbv2 describe-load-balancers --query 'LoadBalancers[?DNSName==`'"$SHARED_ALB_URL"'`].LoadBalancerArn' --output text)
echo "The shared ALB is available at: http://$SHARED_ALB_URL"

The ALB load balancer creation process can take a couple of minutes, including its DNS propagation, before you will be able to test it.

➤ 6. Access the application via the shared ALB URL, where you should expect an HTML output from the / path (coming from the ui component):

curl -s "http://$SHARED_ALB_URL/"

➤ 7. Access the the /catalog/products path, where you should expect a JSON response with a list of catalog items:

curl -s "http://$SHARED_ALB_URL/catalog/products" | jq

Summary

In this lab, we've learned how to:

    Deploy a real-world microservices application on EKS Auto Mode
    Set up ALB ingress for both frontend and backend services
    Use path-based routing to organize our microservices
    Observe load balancing across multiple pods
