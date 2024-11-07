## Install Velero Client
- Documentation
    - Velero Installation : https://velero.io/docs/v1.15/basic-install/
    - Velero AWS Providers: https://velero.io/docs/v1.8/supported-providers/

### From Github
git clone https://github.com/vmware-tanzu/velero

git clone 

curl -LO https://github.com/vmware-tanzu/velero/releases/download/v1.15.0/velero-v1.15.0-linux-amd64.tar.gz

curl -LO https://github.com/vmware-tanzu/velero/releases/download/v1.15.0/velero-v1.15.0-linux-arm.tar.gz

tar -C /usr/local/bin -xzvf velero-v1.15.0-linux-amd64.tar.gz

sudo tar -C /usr/local/bin -xzvf velero-v1.15.0-linux-arm.tar.gz

export PATH=$PATH:/usr/local/bin/velero-v1.15.0-linux-amd64/

export PATH=$PATH:/usr/local/bin/velero-v1.15.0-linux-arm/

### Using Homebrew
brew install velero

whereami velero

export PATH=$PATH:/opt/homebrew/bin/velero

velero version


## Create S3 bucket

BUCKET=<YOUR_BUCKET>
REGION=<YOUR_REGION>
aws s3api create-bucket \
    --bucket $BUCKET \
    --region $REGION \
    --create-bucket-configuration LocationConstraint=$REGION+


aws s3api create-bucket \
    --bucket $BUCKET \
    --region us-east-1

## Create IAM user

### Set permissions for Velero

- Create IAM User
aws iam create-user --user-name velero

If you'll be using Velero to backup multiple clusters with multiple S3 buckets, it may be desirable to create a unique username per cluster rather than the default velero.

- Attach policies to give velero the necessary permissions (note that s3:PutObjectTagging is only needed if you make use of the config.tagging field in the BackupStorageLocation spec):

cat > velero-policy.json <<EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeVolumes",
                "ec2:DescribeSnapshots",
                "ec2:CreateTags",
                "ec2:CreateVolume",
                "ec2:CreateSnapshot",
                "ec2:DeleteSnapshot"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:PutObject",
                "s3:PutObjectTagging",
                "s3:AbortMultipartUpload",
                "s3:ListMultipartUploadParts"
            ],
            "Resource": [
                "arn:aws:s3:::${BUCKET}/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::${BUCKET}"
            ]
        }
    ]
}
EOF

- Attach Policy

aws iam put-user-policy \
  --user-name velero \
  --policy-name velero \
  --policy-document file://velero-policy.json

- Create an access key for the user:
aws iam create-access-key --user-name velero

- Create a Velero-specific credentials file (credentials-velero) in your local directory:

[default]
aws_access_key_id=<AWS_ACCESS_KEY_ID>
aws_secret_access_key=<AWS_SECRET_ACCESS_KEY>

### Install Velero Server
velero install  \
    --provider aws \
    --bucket velero \
    --secret-file ./credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3url=http://minio.velero.svc:9000

velero install  \
   --provider aws  \
   --plugins velero/velero-plugin-for-aws:v1.11.0  \
   --bucket $BUCKET  \
   --prefix backups/cluster1   \
   --backup-location-config region=eu-west-1    \ 
   --snapshot-location-config region=eu-west-1   \  
   --secret-file ./credentials-velero  


velero version

kubectl apply -f nginx-app.yaml  --> probably for testing purpose, to have somekind of deplyoment

kubectl get deployments -n nginx-example

kubectl get ns

kubectl get deployments -l component=velero --namespace=velero

### Take backup resources
velero backup create <backup-name> --include-namespaces <namespace> --snapshot-volumes
    - backup-name: Give your backup a unique name.
    - namespace: Specify the namespace where your PVC resides.
    - --snapshot-volumes: This option tells Velero to create snapshots of the Persistent Volumes associated with PVCs in the namespace.


velero backup create my-app-backup \
  --include-namespaces my-app \
  --include-resources persistentvolumeclaims,persistentvolumes

#### To back up both the metadata and the data, you can use both flags together:
velero backup create helix-infra-backup --include-namespaces helix-infra --include-resources persistentvolumeclaims,persistentvolumes --snapshot-volumes


velero backup create nginx-example-backup --include-namespaces nginx-example --include-resources persistentvolumeclaims,persistentvolumes --snapshot-volumes



### Verify Backup
velero backup describe <backup-name> --details

kubectl delete namespace nginx-example

kubectl get deployment --namespace=nginx-example

kubectl get services --namespace=nginx-example

kubectl get namespace/nginx-example


### Restore PVCs and PCs (if needed)
velero restore create --from-backup nginx-example-backup

velero restore create --from-backup nginx-backup

velero restore get

kubectl get services --namespace=nginx-example

kubectl get namespace/nginx-example



---------------------------------------------
BUCKET=pvc-backup-test-velero
REGION=eu-west-1


aws iam create-access-key --user-name velero > /tmp/key.json