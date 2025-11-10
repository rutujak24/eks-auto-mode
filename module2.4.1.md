StatefulSets and PersistentVolumes

Overview | Inspect SQL databases | Demonstrate the ephemeral nature of emptyDir | Summary
Overview

The catalog microservice utilizes an SQL database running in a separate Pod in the cluster. Before we dive deeper into how these are deployed, let's review a number of relevant Kubernetes concepts:

    A Volume 

is a data store which is accessible to the containers in a pod. How that data store comes to be, and the medium that backs it, are determined by the particular volume type used.
An Ephemeral Volume 
follows a pod's lifetime, and gets created and deleted along with the pod.
A PersistentVolume 
(PV) is a piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned using a StorageClass. It is a resource in the cluster just like a node is a cluster resource. PVs are a special kind of Volume with a lifecycle independent of any individual Pod that uses the PV.
A PersistentVolumeClaim 
(PVC) is a request for persistent storage by a user. It is the storage equivalent of a pod, wherein pods consume node resources and PVCs consume PV resources. Just as pods can request specific levels of resources (CPU and Memory), claims can request specific size and access modes (e.g. they can be mounted ReadWriteOnce, ReadOnlyMany, ReadWriteMany, or ReadWriteOncePod).
A StatefulSet 
runs a group of pods, and maintains a sticky identity for each of those pods. Although individual pods in a StatefulSet are susceptible to failure, the persistent Pod identifiers make it easier to match existing volumes to the new pods that replace any that have failed. This is useful for managing applications, such as databases, that need persistent storage.
A StorageClass 
provides a way for administrators to describe a "class" of storage they offer. Different classes might map to quality-of-service levels, or to backup policies, or to arbitrary policies determined by cluster administrators.
Dynamic Volume Provisioning 

    allows storage volumes to be created on-demand. Without dynamic provisioning, cluster administrators have to manually make calls to their cloud or storage provider to create new storage volumes, and then create PersistentVolume objects to represent them in Kubernetes. The dynamic provisioning feature eliminates the need for cluster administrators to pre-provision storage. Instead, it automatically provisions storage when it is requested by users.

Enable MySQL component for the catalog service

Since we initially deployed the catalog service with an in-memory database, let's first enable a MySQL Pod with no persistent storage, before demonstrating how to use Amazon EKS Auto Mode for stateful applications. Run the below command:

helm upgrade -i retail-store-app-catalog oci://public.ecr.aws/aws-containers/retail-store-sample-catalog-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} -f - <<EOF
mysql:
  create: true
EOF

Ensure that the MySQL Pod of the catalog service is up and running:

kubectl get pods -l app.kubernetes.io/instance=retail-store-app-catalog -l app.kubernetes.io/component=mysql

Inspect SQL databases

The catalog microservice utilizes a MySQL database running in a Pod in the cluster.

➤ Let's inspect the MySQL database Pod to see its current volume configuration:

kubectl describe statefulset retail-store-app-catalog-mysql

You should see output similar to the below:

Name:               retail-store-app-catalog-mysql-0
Namespace:          default
...
Replicas:           1 desired | 1 total
...
Pod Template:
...
  Containers:
   mysql:
    Image:      public.ecr.aws/docker/library/mysql:8.0
    Port:       3306/TCP
    Host Port:  0/TCP
    Environment:
      MYSQL_ROOT_PASSWORD:  my-secret-pw
      MYSQL_DATABASE:       catalog
      MYSQL_USER:           <set to the key 'username' in secret 'catalog-db'> Optional: false
      MYSQL_PASSWORD:       <set to the key 'password' in secret 'catalog-db'>  Optional: false
    Mounts:
      /var/lib/mysql from data (rw)
  Volumes:
   data:
    Type:          EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:     <unset>
  Node-Selectors:  <none>
  Tolerations:     <none>
Volume Claims:     <none>
...

We can make the following observations:

    The MySQL database is deployed as a StatefulSet with a single replica.
    The Pod template includes a single mysql container, with a data volume of type emptyDir.

Demonstrate the ephemeral nature of emptyDir

An emptyDir volume is first created when a Pod is assigned to a node, and exists as long as that Pod is running on that node. As the name implies, the emptyDir volume is initially empty. All containers in the Pod can read and write the same files in the emptyDir volume, though that volume can be mounted on the same or different paths in each container. When a Pod is removed from a node for any reason, the data in the emptyDir is deleted permanently. Therefore emptyDir is not a good fit for our SQL databases.

We can demonstrate the ephemeral nature of emptyDir by starting a shell session inside the MySQL container and creating a test file. After that we'll delete the Pod that is running in our StatefulSet. Because the Pod is using an emptyDir and not a PV, the file will not survive a Pod restart.

➤ First let's run a command inside our MySQL container to create a file in the /tmp directory:

kubectl exec retail-store-app-catalog-mysql-0 -- bash -c "echo 123 > /tmp/test.txt"

➤ Now, let's verify our test.txt file was created in the tmp directory:

kubectl exec retail-store-app-catalog-mysql-0 -- cat /tmp/test.txt

You should see the contents of the file we created:

123

➤ Now, let's remove the current retail-store-app-catalog-mysql Pod to force the StatefulSet controller to automatically re-create a new retail-store-app-catalog-mysql Pod:

kubectl delete pod retail-store-app-catalog-mysql-0

➤ Wait for the Pod to be re-created:

kubectl wait --for=condition=Ready pod retail-store-app-catalog-mysql-0 --timeout=30s

After a few seconds you should see:

pod/retail-store-app-catalog-mysql-0 condition met

➤ Verify the Pod is running:

kubectl get pod retail-store-app-catalog-mysql-0

The output should be similar to the following:

NAME                                   READY   STATUS    RESTARTS   AGE
retail-store-app-catalog-mysql-0   1/1     Running   0          48s

➤ Check for the presence of test.txt in the /tmp directory:

kubectl exec retail-store-app-catalog-mysql-0 -- cat /tmp/test.txt

With the following output:

cat: /tmp/test.txt: No such file or directory
command terminated with exit code 1

You can see that the test.txt file no longer exists due to emptyDir volumes being ephemeral.
Summary

In this section, we performed the following steps:

    Inspected the StatefulSet resources associated with the catalog MySQL database.
    Verified that this database is provisioned using an emptyDir volume.
    Demonstrated the ephemeral nature of emptyDir volumes.

In the next section, we will define a default StorageClass and use this to create a persistent volume for the catalog MySQL database.
