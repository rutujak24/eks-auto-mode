Multiple EBS Storage Classes

Overview | Update StatefulSet configuration | Summary
Overview

In the previous section we created a default StorageClass and then configured the catalog service to use this to create a PV for its MySQL database.

The default StorageClass we created uses a KMS key to encrypt volumes, and also uses the default gp3 setting for IOPS (which we verified to be 3000).

Now suppose we were expecting a higher volume of traffic on the orders database, and needed to increase the IOPS accordingly.

In this section, we will create a second StorageClass that provisions an encrypted gp3 volume with 6000 IOPS, and then explicitly configure the orders PostgreSQL StatefulSet to use a PV based on this StorageClass.
Update StatefulSet configuration
Create a second StorageClass with 6000 IOPS

➤ Create a StorageClass using the same KMS key from the previous section, but also specifying an IOPS value:

cat >~/environment/ebs-iops-kms-sc.yaml <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: eks-auto-ebs-iops-kms-sc
provisioner: ebs.csi.eks.amazonaws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  iops: "6000"
  encrypted: "true"
  kmsKeyId: $KEY_ID
EOF

kubectl apply -f ~/environment/ebs-iops-kms-sc.yaml

Note that this time we did not include an annotation to make this the default StorageClass. This means we will need to reference this StorageClass explicitly in order to use it.
Update the orders PostgreSQL database pod

Since we can't update some of the StatefulSet fields such as the persistentVolumeClaimRetentionPolicy, we will first have to uninstall the current version of the orders service, and reinstall it again.

➤ Execute the following command:

helm uninstall retail-store-app-orders

➤ Create a values-orders.yaml and redeploy the orders chart as follows:

helm upgrade -i retail-store-app-orders oci://public.ecr.aws/aws-containers/retail-store-sample-orders-chart \
  --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} -f - <<EOF
app:
  persistence:
    provider: postgres
    endpoint: ""
    database: "orders"

    secret:
      create: true
      name: orders-db
      username: orders
      password: "postgres123"

postgresql:
  create: true
  persistentVolume:
    enabled: true
    accessMode:
      - ReadWriteOnce
    size: 20Gi
    storageClass: eks-auto-ebs-iops-kms-sc
EOF

➤ Wait until the orders service is up and running again (it may restart and take a minute):

kubectl wait --for=condition=Ready pod -l app.kubernetes.io/instance=retail-store-app-orders --namespace default --timeout=300s

Verify that the PVC for the orders PostgreSQL database has been created

➤ List all PVCs:

kubectl get pvc

You should see:

NAME                                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS               VOLUMEATTRIBUTESCLASS   AGE
data-retail-store-app-catalog-mysql-0       Bound    pvc-c5c4dec1-0ce5-4a00-982d-233c7d5bfdbb   30Gi       RWO            eks-auto-ebs-kms-sc        <unset>                 93m
data-retail-store-app-orders-postgresql-0   Bound    pvc-b1e886af-a019-4d98-adb0-d321e5dcab80   20Gi       RWO            eks-auto-ebs-iops-kms-sc   <unset>                 6m55s

data-retail-store-app-orders-postgresql-0 is the PVC created for the PostgreSQL DB.

➤ Inspect it using:

kubectl describe pvc data-retail-store-app-orders-postgresql-0

➤ You can also inspect the PV as follows:

kubectl describe pv $(kubectl get pvc data-retail-store-app-orders-postgresql-0 -o jsonpath="{.spec.volumeName}")

Verify that the EBS volume for the orders PostgreSQL database has been created correctly

Let us verify that the IOPS attribute has been set correctly.

➤ Obtain the underlying AWS EBS Volume ID as follows:

PGSQL_PV_NAME=$(kubectl get pvc data-retail-store-app-orders-postgresql-0 -o jsonpath="{.spec.volumeName}")
PGSQL_EBS_VOL_ID=$(kubectl get pv $PGSQL_PV_NAME -o jsonpath="{.spec.csi.volumeHandle}")
echo EBS Volume ID: $PGSQL_EBS_VOL_ID

➤ Display the details for the EBS volume:

aws ec2 describe-volumes --volume-ids $PGSQL_EBS_VOL_ID --query Volumes[0]

Note the following sections of the output, proving that IOPS is set to 6000 and KMS encryption is enabled with the correct key:

    ...
    "Encrypted": true,
    "KmsKeyId": "arn:aws:kms:us-west-2:845041152230:key/2d61cc69-7f98-474d-a663-b682872a9f6a",
    "Size": 20,
    ...
    "Iops": 6000,
    ...

Summary

In this section, we performed the following steps:

    Created a StorageClass configured to use a KMS key for encryption, and using a standard gp3 volume with 6000 provisioned IOPS.
    Updated the configuration of the orders service to use this new StorageClass for creating a persistent volume.
    Verified that the EBS volume was created correctly, using the correct KMS key and also the specified provisioned IOPS setting.
