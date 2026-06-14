---
myst:
  html_meta:
    description: "How to deploying on aws on public clouds with Livepatch server."
---


(livepatch_server_on_public_clouds-how-to-guides-deploying-the-livepatch-server-snap-on-aws)=

# Deploying the Livepatch Server Snap on AWS

This section details how to deploy the Livepatch Server snap in auto-scaling configuration on Azure.

## Required Resources

To set up Livepatch server on AWS using an S3 bucket for patch storage, you will need:

- An RDS instance with PostgreSQL 12 or 14 using password authentication.
  - DSN string with User and password should be stored in the AWS secrets vault.
- A Launch template with an instance type of at least T3.Medium (2 vCPU, 4 GB RAM)
- An Ubuntu Pro token stored in the AWS secrets vault.
- S3 bucket with the `s3:GetObject` policy (bucket must be public for clients to download patches, but you can restrict the policy to a range of ip addresses).
- An IAM role with the `AWSSecretsManagerClientReadOnlyAccess` role and `AmazonS3ReadOnlyAccess` role.
- An IAM user with Allow effects for the actions `s3:PutObject`, `s3:ListBucket`, and `s3:GetObject` roles for the S3 bucket.
- A secret entry for the IAM user with Key value pairs `S3AccessKey` (for the S3 access key), and `SecretAccessKey` (for the S3 secret key).
- A Load balancer set to internet-facing.
- An auto-scaling group using the launch template.
- A security group to allow all internet access for HTTP and HTTPS.
- A security group to allow access to only traffic from within AWS (for the server instances and PostgreSQL).

## Creating The Deployment

With AWS, you can set up an auto-scaling group and a load balancer with an EC2 launch template configured to install and set up a Livepatch server instance.

To create a launch template, log in to the AWS console and navigate to the Launch Templates section in the EC2 overview page.

You want to select an instance type that has sufficient CPU and memory capacity. The minimum instance type for Livepatch server to run efficiently is the t3.medium instance type, with 2 vCPU and 4 GB memory. For the storage options, the default volume size of 8 GB will be sufficient.

For the network settings, select a security group that only allows network traffic from within AWS. We will set up a load balancer with internet access and redirect to our server instances later. For debugging purposes, you can create an SSH Key Pair to connect to the instance.

In the advanced details section, set the IAM instance profile to the profile with the `AWSSecretsManagerClientReadOnlyAccess` role and `AmazonS3ReadOnlyAccess` role. Also in the Advanced Details section, scroll down to the user data field. This is where you can upload or copy and paste the cloud-init module. The following example shows the cloud-init template from earlier configured to run on AWS using S3 buckets as patch storage.

```yaml
#cloud-config
package_update: true
package_upgrade: true
packages:
  - snapd
  - jq

write_files:
  - path: /etc/client/Livepatch.env
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
      export URLTEMPLATE=https://<bucket name>.s3-<region>.amazonaws.com/{filaname}
  - path: /etc/client/database_url
    permissions: "0600"
    owner: root:root
  - path: /etc/client/pro_token
    permissions: "0600"
    owner: root:root
  - path: /etc/client/livepatch_admin_user
    permissions: "0600"
    owner: root:root
  - path: /etc/client/s3
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
    . /etc/client/Livepatch.env        
    # setup PostgreSQL connection  
    aws secretsmanager get-secret-value \
     --secret-id $DB_SECRET_ID \
     --query SecretString \
     --output text \
     --region eu-north-1 | \
    jq -r \
     --arg DB_ENDPOINT "$DB_ENDPOINT" \
     --arg PORT "$PORT" \
     '"PostgreSQL://\(.username):\(.password | @uri)@\($DB_ENDPOINT):\($PORT)"' > /etc/client/database_url
    
    # set pro token
    aws secretsmanager get-secret-value \
    --secret-id $TOKEN_SECRET_ID \
    --query SecretString \
    --region eu-north-1 > /etc/client/pro_token
    
    # setup S3 storage
    aws secretsmanager get-secret-value \
        --secret-id $S3_SECRET_ID \
        --query SecretString \
        --region eu-north-1 \
        --output text > /etc/client/s3

    # setup user
    aws secretsmanager get-secret-value \
        --secret-id $ADMIN_USER_SECRET_ID \
        --query SecretString \
        --region eu-north-1 \
        --output text > /etc/client/livepatch_admin_user

  - |
    set -e
    snap stop canonical-livepatch-server.livepatch
    canonical-livepatch-server.schema-tool "$(cat /etc/client/database_url)"

    snap set canonical-livepatch-server \
    token="$(cat /etc/client/pro_token)" \
    lp.database.connection-string="$(cat /etc/client/database_url)" \
    lp.patch-storage.type=s3 \
    lp.server.url-template=$URLTEMPLATE \
    lp.patch-storage.s3-bucket="$S3BUCKET" \
    lp.patch-storage.s3-endpoint="$S3ENDPOINT" \
    lp.patch-storage.s3-region="$S3REGION" \
    lp.patch-storage.s3-secure=true \
    lp.patch-storage.s3-access-key="$(cat /etc/client/s3 | jq -r '.S3AccessKey' )" \
    lp.patch-storage.s3-secret-key="$(cat /etc/client/s3 | jq -r '.SecretAccessKey' )" \
    lp.auth.basic.enabled=true \
    lp.patch-sync.enabled=true \
    lp.patch-sync.interval=12h \
    lp.auth.basic.users="$(cat /etc/client/livepatch_admin_user)" \
    lp.server.server-address=0.0.0.0:80
  - |
    rm /etc/client/*

final_message: The system is up, up to date, and Livepatch server is active after $UPTIME second
```

Where the blanked out values will be where you put your relevant resource information. For this template, the S3 and database secrets are assumed to be stored in a JSON object, while the Ubuntu Pro token and admin user string are plaintext.

Once you have a launch template created, test the deployment by going to **Launch Instances** and then to **Launch Instance from Template**. On the launch page, make sure to set the template version to the correct version if you have multiple versions. The instance will take a few minutes to create and configure Livepatch server. Once the instance is ready, you can check if the deployment is complete by running:

```bash
sudo snap logs canonical-livepatch-server
```

If there are no error logs, then the server has successfully initialized. If there are errors, check the status of `cloud-init` with:

```bash
cloud-init status --long
```

If the status shows: **error**, then something went wrong during the cloud-init procedure. You can view the command output logs at `/var/log/cloud-init-output.log`.

With the server launch template ready, next navigate to the auto-scaling groups section and head to **Create Auto Scaling Group**. Give it a name and select the launch template we just created in the previous step. For optimal availability, set the number of instances to be between two and four.

Next, select the region and availability zones you want instances to be created in. Give the auto-scaling group a security group that has access to inside AWS (for the server instances), but allow all HTTP and HTTPS traffic so a load balancer can route incoming traffic. The default VPC will suffice for a multi-unit deployment.

If you already have a load balancer set up, you can link the auto-scaling group to it. If not, select Attach to a new load balancer. Set the default routing (forward to) option to Create a target group, which will set the load balancer to forward traffic to all instances created by the auto scaling group.

On the next page, configure the group size and scaling options, and optionally an automatic scaling policy and maintenance policy.

Once the auto-scaling group is created, it will begin to provision Livepatch server instances based on the given launch template. You can then log in with the admin tool (assuming you defined an admin user in the cloud-init config) by setting the endpoint URL to the public URL of the load balancer. Make sure the security settings for the load balancer allow external traffic so you can login and run administrative duties with the admin tool.
