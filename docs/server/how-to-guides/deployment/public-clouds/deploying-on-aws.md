---
myst:
  html_meta:
    description: "How to deploy the Livepatch server snap on AWS."
---


(server-how-to-guides-deploying-the-livepatch-server-snap-on-aws)=

# How to deploy on AWS

This section details how to deploy the Livepatch Server snap in auto-scaling configuration on AWS.

## Required resources

To set up the Livepatch server on AWS using an S3 bucket for patch storage, the following are required:

* An RDS instance with PostgreSQL 12 or 14 using password authentication. The DSN string with username and password should be stored in the AWS Secrets Vault.
* A launch template with an instance type of at least T3.Medium (two vCPUs, 4 GB RAM).
* An Ubuntu Pro token stored in the AWS Secrets Vault.
* An S3 bucket with the `s3:GetObject` policy. The bucket must be public for clients to download patches, but the policy can be restricted to a range of IP addresses.
* An IAM role with the `AWSSecretsManagerClientReadOnlyAccess` role and `AmazonS3ReadOnlyAccess` role.
* An IAM user with Allow effects for the actions `s3:PutObject`, `s3:ListBucket`, and `s3:GetObject` roles for the S3 bucket.
* A secret entry for the IAM user with key-value pairs `S3AccessKey` (for the S3 access key) and `SecretAccessKey` (for the S3 secret key).
* A load balancer set to internet-facing.
* An auto-scaling group using the launch template.
* A security group to allow all internet access for HTTP and HTTPS.
* A security group to allow access only from traffic within AWS (for the server instances and PostgreSQL).

## Create the deployment

With AWS, an auto-scaling group and a load balancer can be set up with an EC2 launch template configured to install and set up a Livepatch server instance.

### Create the launch template

To create a launch template, log in to the AWS console and navigate to the Launch Templates section in the EC2 overview page.

Select an instance type with sufficient CPU and memory capacity. The minimum instance type for the Livepatch server to run efficiently is `t3.medium`, with two vCPUs and 4 GB memory. For storage options, the default volume size of 8 GB is sufficient.

For network settings, select a security group that only allows network traffic from within AWS. Set up a load balancer with internet access later to redirect traffic to the server instances. For debugging purposes, create an SSH key pair to connect to the instance.

In the Advanced Details section, set the IAM instance profile to the profile with the `AWSSecretsManagerClientReadOnlyAccess` role and `AmazonS3ReadOnlyAccess` role. Also in the Advanced Details section, scroll down to the User Data field to upload or paste the cloud-init module. The following example shows the cloud-init template configured to run on AWS using S3 buckets as patch storage:

```yaml
#cloud-config
package_update: true
package_upgrade: true
packages:
  - snapd
  - jq

write_files:
  - path: /etc/livepatch/Livepatch.env
    permissions: '0600'
    owner: root:root
    content: |
      # database variables 
      export DB_ENDPOINT=<db endpoint>
      export PORT=5432
      export DB_SECRET_ID=<db secret id>

      # pro token variables
      export TOKEN_SECRET_ID=<token secret id>
      export ADMIN_USER_SECRET_ID=<admin user secret id>
      
      # S3 variables
      export S3_SECRET_ID=<s3 secret id>
      export S3BUCKET=<bucket name>
      export S3ENDPOINT=<bucket endpoint>
      export S3REGION=<region> 
      export URLTEMPLATE=https://<bucket name>.s3-<region>.amazonaws.com/{filename}
  - path: /etc/livepatch/database_url
    permissions: "0600"
    owner: root:root
  - path: /etc/livepatch/pro_token
    permissions: "0600"
    owner: root:root
  - path: /etc/livepatch/livepatch_admin_user
    permissions: "0600"
    owner: root:root
  - path: /etc/livepatch/s3
    permissions: "0600"
    owner: root:root

snap:
  commands:
    - snap install aws-cli --classic
    - snap install canonical-livepatch-server
    - snap refresh --hold canonical-livepatch-server

runcmd:
  - |
    set -e
    snap wait system seed.loaded
    . /etc/livepatch/Livepatch.env        
    # setup PostgreSQL connection  
    aws secretsmanager get-secret-value \
     --secret-id $DB_SECRET_ID \
     --query SecretString \
     --output text \
     --region eu-north-1 | \
    jq -r \
     --arg DB_ENDPOINT "$DB_ENDPOINT" \
     --arg PORT "$PORT" \
     '"PostgreSQL://\(.username):\(.password | @uri)@\($DB_ENDPOINT):\($PORT)"' > /etc/livepatch/database_url
    
    # set pro token
    aws secretsmanager get-secret-value \
    --secret-id $TOKEN_SECRET_ID \
    --query SecretString \
    --region eu-north-1 > /etc/livepatch/pro_token
    
    # setup S3 storage
    aws secretsmanager get-secret-value \
        --secret-id $S3_SECRET_ID \
        --query SecretString \
        --region eu-north-1 \
        --output text > /etc/livepatch/s3

    # setup user
    aws secretsmanager get-secret-value \
        --secret-id $ADMIN_USER_SECRET_ID \
        --query SecretString \
        --region eu-north-1 \
        --output text > /etc/livepatch/livepatch_admin_user

  - |
    set -e
    snap stop canonical-livepatch-server.livepatch
    canonical-livepatch-server.schema-tool "$(cat /etc/livepatch/database_url)"

    snap set canonical-livepatch-server \
    token="$(cat /etc/livepatch/pro_token)" \
    lp.database.connection-string="$(cat /etc/livepatch/database_url)" \
    lp.patch-storage.type=s3 \
    lp.server.url-template=$URLTEMPLATE \
    lp.patch-storage.s3-bucket="$S3BUCKET" \
    lp.patch-storage.s3-endpoint="$S3ENDPOINT" \
    lp.patch-storage.s3-region="$S3REGION" \
    lp.patch-storage.s3-secure=true \
    lp.patch-storage.s3-access-key="$(cat /etc/livepatch/s3 | jq -r '.S3AccessKey' )" \
    lp.patch-storage.s3-secret-key="$(cat /etc/livepatch/s3 | jq -r '.SecretAccessKey' )" \
    lp.auth.basic.enabled=true \
    lp.patch-sync.enabled=true \
    lp.patch-sync.interval=12h \
    lp.auth.basic.users="$(cat /etc/livepatch/livepatch_admin_user)" \
    lp.server.server-address=0.0.0.0:80
  - |
    rm /etc/livepatch/*

final_message: The system is up, up to date, and Livepatch server is active after $UPTIME second
```

Replace the blanked-out values with the relevant resource information. For this template, the S3 and database secrets are assumed to be stored in a JSON object, while the Ubuntu Pro token and admin user string are plaintext.

### Test the deployment

Once a launch template is created, test the deployment by going to **Launch Instances** and then to **Launch Instance from Template**. On the launch page, ensure the template version is set to the correct version if multiple versions exist. The instance takes a few minutes to create and configure the Livepatch server. Once the instance is ready, check if the deployment is complete by running:

```bash
sudo snap logs canonical-livepatch-server
```

If there are no error logs, the server has initialized successfully. If there are errors, check the status of `cloud-init` with:

```bash
cloud-init status --long
```

If the status shows **error**, something went wrong during the cloud-init procedure. The command output logs can be viewed at `/var/log/cloud-init-output.log`.

### Create the auto-scaling group

With the server launch template ready, navigate to the Auto Scaling Groups section and select **Create Auto Scaling Group**. Provide a name and select the launch template created in the previous step. For optimal availability, set the number of instances to between two and four.

Next, select the region and availability zones where instances should be created. Assign a security group that has access to inside AWS (for the server instances), but allow all HTTP and HTTPS traffic so a load balancer can route incoming traffic. The default VPC is sufficient for a multi-unit deployment.

If a load balancer is already set up, the auto-scaling group can be linked to it. If not, select **Attach to a new load balancer**. Set the default routing (forward to) option to **Create a target group**, which configures the load balancer to forward traffic to all instances created by the auto-scaling group.

On the next page, configure the group size and scaling options, and optionally an automatic scaling policy and maintenance policy.

Once the auto-scaling group is created, it begins provisioning Livepatch server instances based on the given launch template. The admin tool can then be used to log in (assuming an admin user was defined in the cloud-init configuration) by setting the endpoint URL to the public URL of the load balancer. Ensure the security settings for the load balancer allow external traffic so administrative duties can be performed with the admin tool.
