Default Storage Class using EBS CSI driver

Overview | Update the components | Summary
Overview

In the previous section we saw that the databases for the catalog and orders services are using ephemeral emptyDir volumes for storage. In the remainder of this module, we will create a StorageClass which we will then use to replace these volumes with persistent volumes using the EBS CSI driver. We will also learn how we can use the StorageClass to configure a volume parameter, such as a KMS key to encrypt the volume.

In this section we will create a default StorageClass and use this to create a persistent volume for the catalog MySQL database. In the next section, we will then create an additional StorageClass for the orders PostgreSQL database which increases the amount of provisioned IOPS.
Update the components
Create a default StorageClass with KMS encryption

➤ First, let's create a KMS key as follows:

KEY_ID=$(aws kms create-key --tags TagKey=Name,TagValue=eks-automode-workshop --query 'KeyMetadata.KeyId' --output text)
KEY_ARN=$(aws kms describe-key --key-id $KEY_ID --query 'KeyMetadata.Arn' --output text)
echo "Key Id:" $KEY_ID
echo "Key Arn:" $KEY_ARN

➤ Next, let's create an IAM resource policy JSON document for the KMS key, which allows the CSI service that runs on the EC2 managed instance to assume that role to encrypt & decrypt the data written to the EBS volume:

cat >key-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Id": "key-auto-policy-3",
    "Statement": [
        {
            "Sid": "iam-kms",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::$AWS_ACCOUNT_ID:root"
            },
            "Action": "kms:*",
            "Resource": "*"
        },
        {
            "Sid": "ec2-kms",
            "Effect": "Allow",
            "Principal": {
                "AWS": "*"
            },
            "Action": [
                "kms:Encrypt",
                "kms:Decrypt",
                "kms:ReEncrypt*",
                "kms:GenerateDataKey*",
                "kms:CreateGrant",
                "kms:DescribeKey"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "kms:CallerAccount": "$AWS_ACCOUNT_ID",
                    "kms:ViaService": "ec2.$AWS_REGION.amazonaws.com"
                }
            }
        }
    ]
}
EOF

➤ Now let's attach this policy document to the KMS key:

aws kms put-key-policy --key-id $KEY_ID --policy file://key-policy.json

➤ Finally, let's create a new storage class using the KMS key:

cat >~/environment/ebs-kms-sc.yaml <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: eks-auto-ebs-kms-sc
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.eks.amazonaws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  encrypted: "true"
  kmsKeyId: $KEY_ID
EOF

kubectl apply -f ~/environment/ebs-kms-sc.yaml

➤ Let's inspect the StorageClass we just created:

kubectl describe storageclass eks-auto-ebs-kms-sc

This should produce the following output:

Name:            eks-auto-ebs-kms-sc
IsDefaultClass:  Yes
Annotations:     kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"},"name":"eks-auto-ebs-kms-sc"},"parameters":{"encrypted":"true","kmsKeyId":"2d61cc69-7f98-474d-a663-b682872a9f6a","type":"gp3"},"provisioner":"ebs.csi.eks.amazonaws.com","volumeBindingMode":"WaitForFirstConsumer"}
,storageclass.kubernetes.io/is-default-class=true
Provisioner:           ebs.csi.eks.amazonaws.com
Parameters:            encrypted=true,kmsKeyId=2d61cc69-7f98-474d-a663-b682872a9f6a,type=gp3
AllowVolumeExpansion:  <unset>
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     WaitForFirstConsumer
Events:                <none>

We can make the following observations:

    eks-auto-ebs-kms-sc is configured as the default storage class.
    The associated provisioner is ebs.csi.eks.amazonaws.com. This provisioner is automatically made available by EKS Auto Mode.
    The EBS volume type is set to gp3 (defaults to 3000 IOPS).
    The ReclaimPolicy is set to Delete, which means that when the associated PVC is deleted, this results in the deletion of both the PV object in Kubernetes, as well as the associated storage asset in the external infrastructure.
    The VolumeBindingMode 

    is set to WaitForFirstConsumer. This mode delays the binding and provisioning of a PersistentVolume until a Pod using the PersistentVolumeClaim is created. PersistentVolumes will be selected or provisioned conforming to the topology that is specified by the pod's scheduling constraints. This is needed in situations where storage backends are topology-constrained and not globally accessible from all Nodes in the cluster (as would be the case where multiple AZs are used).
    Encryption is configured for volumes using the KMS key we created.

Update the catalog MySQL database Pod

Now that we have a default StorageClass in place, let's update the catalog service to use it. Since many of StatefulSet fields, including volumeClaimTemplates, cannot be modified, we will uninstall the catalog component and then reinstall it, so we can update the volume type from emptyDir to a Persistent Volume.

➤ First, uninstall the current catalog service:

helm uninstall retail-store-app-catalog

Re-launch the catalog service using:

helm upgrade -i retail-store-app-catalog oci://public.ecr.aws/aws-containers/retail-store-sample-catalog-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} -f - <<EOF
app:
  persistence:
    provider: mysql
    endpoint: ""
    database: "catalog"

    secret:
      create: true
      name: catalog-db
      username: catalog
      password: "mysqlcatalog123"

mysql:
  create: true
  persistentVolume:
    enabled: true
    accessMode:
      - ReadWriteOnce
    size: 30Gi
EOF

Verify that the PVC for the catalog MySQL database has been created

The re-launched catalog service should now have an associated PersistentVolumeClaim.

➤ We can see this by running:

kubectl describe statefulset retail-store-app-catalog-mysql

This time the output shows:

Name:               retail-store-app-catalog-mysql
Namespace:          default
...
  Containers:
   mysql:
    Image:      public.ecr.aws/docker/library/mysql:8.0
    Port:       3306/TCP
    Host Port:  0/TCP
    Environment:
      MYSQL_ROOT_PASSWORD:  my-secret-pw
      MYSQL_DATABASE:       catalog
      MYSQL_USER:           <set to the key 'username' in secret 'catalog-db'>  Optional: false
      MYSQL_PASSWORD:       <set to the key 'password' in secret 'catalog-db'>  Optional: false
    Mounts:
      /var/lib/mysql from data (rw)
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Volume Claims:
  Name:          data
  StorageClass:
  Labels:        <none>
  Annotations:   <none>
  Capacity:      30Gi

Let's inspect the PVC resource.

➤ To list all PVCs use:

kubectl get pvc

You should see:

NAME                                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          VOLUMEATTRIBUTESCLASS   AGE
data-retail-store-app-catalog-mysql-0   Bound    pvc-45a9e6ab-ed7c-47e1-9576-c3f01f33d327   30Gi       RWO            eks-auto-ebs-csi-sc   <unset>                 4h43m

data-retail-store-app-catalog-mysql-0 is the PVC created for the MySQL DB.

➤ Let's inspect it using:

kubectl describe pvc data-retail-store-app-catalog-mysql-0

This should produce the following output:

Name:          data-retail-store-app-catalog-mysql-0
Namespace:     default
StorageClass:  eks-auto-ebs-kms-sc
Status:        Bound
Volume:        pvc-c5c4dec1-0ce5-4a00-982d-233c7d5bfdbb
Labels:        ...
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
               volume.beta.kubernetes.io/storage-provisioner: ebs.csi.eks.amazonaws.com
               volume.kubernetes.io/selected-node: i-04a55c853d8acf24f
               volume.kubernetes.io/storage-provisioner: ebs.csi.eks.amazonaws.com
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      30Gi
Access Modes:  RWO
VolumeMode:    Filesystem
Used By:       retail-store-app-catalog-mysql-0
Events:        <none>

From the output we can see that the PVC is bound to a specific PV (in this case pvc-c5c4dec1-0ce5-4a00-982d-233c7d5bfdbb), and that this has been provisioned using ebs.csi.eks.amazonaws.com with a capacity of 30Gi as specified in values-catalog.yaml.

➤ We can also inspect the PV as follows:

kubectl describe pv $(kubectl get pvc data-retail-store-app-catalog-mysql-0 -o jsonpath="{.spec.volumeName}")

The output should be similar to the below:

Name:              pvc-c5c4dec1-0ce5-4a00-982d-233c7d5bfdbb
Labels:            <none>
Annotations:       pv.kubernetes.io/provisioned-by: ebs.csi.eks.amazonaws.com
                   volume.kubernetes.io/provisioner-deletion-secret-name:
                   volume.kubernetes.io/provisioner-deletion-secret-namespace:
Finalizers:        [external-provisioner.volume.kubernetes.io/finalizer kubernetes.io/pv-protection external-attacher/ebs-csi-eks-amazonaws-com]
StorageClass:      eks-auto-ebs-kms-sc
Status:            Bound
Claim:             default/data-retail-store-app-catalog-mysql-0
Reclaim Policy:    Delete
Access Modes:      RWO
VolumeMode:        Filesystem
Capacity:          30Gi
Node Affinity:
  Required Terms:
    Term 0:        topology.kubernetes.io/zone in [us-west-2a]
Message:
Source:
    Type:              CSI (a Container Storage Interface (CSI) volume source)
    Driver:            ebs.csi.eks.amazonaws.com
    FSType:            ext4
    VolumeHandle:      vol-05938b7ff13ef8ebf
    ReadOnly:          false
    VolumeAttributes:      storage.kubernetes.io/csiProvisionerIdentity=1735365022188-8218-ebs.csi.eks.amazonaws.com
Events:                <none>

The VolumeHandle references the Amazon EBS Volume ID associated with the PV. Let's use this to inspect the EBS volume and check that it has been created correctly.
Verify that the EBS volume for the catalog MySQL database has been created correctly

➤ Obtain the underlying Amazon EBS Volume ID as follows:

MYSQL_PV_NAME=$(kubectl get pvc data-retail-store-app-catalog-mysql-0 -o jsonpath="{.spec.volumeName}")
MYSQL_EBS_VOL_ID=$(kubectl get pv $MYSQL_PV_NAME -o jsonpath="{.spec.csi.volumeHandle}")
echo EBS Volume ID: $MYSQL_EBS_VOL_ID

➤ Display the details for the EBS volume:

aws ec2 describe-volumes --volume-ids $MYSQL_EBS_VOL_ID --query Volumes[0]

Note the following section of the output, proving that KMS encryption is enabled with the correct key:

    ...
    "Encrypted": true,
    "KmsKeyId": "arn:aws:kms:us-west-2:845041152230:key/2d61cc69-7f98-474d-a663-b682872a9f6a",
    ...
    "Iops": 3000,
    ...

Also note that the Iops attribute is set to the gp3 default of 3000.
Summary

In this section, we performed the following steps:

    Created a KMS key.
    Attached a key policy that enables EC2 instances in the account to use the KMS key for encrypting EBS volumes.
    Created a default StorageClass configured to use this key for encryption, and using a standard gp3 volume.
    Updated the configuration of the catalog service to use the default StorageClass for creating a persistent volume.
    Verified that the EBS volume was created correctly, using the correct KMS key and also the default IOPS setting for gp3 volumes.

In the next section we will add a second StorageClass which provisions higher IOPS for gp3 volumes, and explicitly configures the orders PostgreSQL database to use this.
