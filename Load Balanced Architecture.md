# Load Balanced Architecture with Advanced Request Routing (AWS CLI Edition)

Build a Red and a Blue website on EC2, put them behind an Application Load Balancer (ALB), and route traffic by **URL path** (`/red`, `/blue`) and then by **host header** (`red.yourdomain.com`, `blue.yourdomain.com`). Everything is done with the AWS CLI except where noted.

```
                         ┌────────────────────── us-east-1a ──────────────┐
Browser ──> ALB ──rules──┤  Red EC2 (Red target group)                    │
 (HTTP:80)               └────────────────────────────────────────────────┘
                         ┌────────────────────── us-east-1b ──────────────┐
                         │  Blue EC2 (Blue target group)                  │
                         └────────────────────────────────────────────────┘
```
<img width="2720" height="2760" alt="alb_path_host_routing_architecture" src="https://github.com/user-attachments/assets/6d122652-1713-45d3-8efa-6e8ccde3de8b" />



## Contents

1. [Prerequisites](#prerequisites)
2. [Exercise 1: Create the Red and Blue EC2 instances](#exercise-1-create-the-red-and-blue-ec2-instances)
3. [Exercise 2: Path-based routing](#exercise-2-path-based-routing)
4. [Exercise 3: Host-based routing](#exercise-3-host-based-routing)
5. [Exercise 4: Clean up](#exercise-4-clean-up)
6. [Debugging and roadblocks](#debugging-and-roadblocks)

## Prerequisites

| Requirement | Notes |
|---|---|
| AWS CLI v2 configured | `aws sts get-caller-identity` must work |
| IAM permissions | EC2, ELBv2, IAM, S3, Route 53, SSM (read) |
| Course zip extracted | `advanced-request-routing-code.zip` in the current folder |
| Public domain in Route 53 | Needed for Exercise 3 only. **Console task:** registering a domain (course Section 8) is easiest in the console |
| Bash or zsh | Commands use shell variables |

### Files expected from the zip

Your folder should contain the website files (`hw-red.css`, `hw-red-py.css`, `python.png`, `apache.svg`, `red-index.html`, `red-root-index.html` and the blue equivalents) plus `user-data-red` and `user-data-blue`. Open the two user data files and note the exact filenames they download from S3; these are what must be in the bucket.

### Set your variables (run once per terminal session)

Keep these in a file such as `env.sh` and `source env.sh` whenever you open a new terminal.

```bash
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# CHANGE THESE
export BUCKET=arr-bucket-$RANDOM$RANDOM     # must be globally unique
export DOMAIN=yourdomainname.com            # only needed for Exercise 3

echo "Account: $ACCOUNT_ID  Bucket: $BUCKET"
```

Later steps add more variables (`SG_ID`, `RED_ID`, `ALB_ARN` and so on). **Append each one to `env.sh`** so you can resume after closing the terminal.

---

## Exercise 1: Create the Red and Blue EC2 instances

### Task 1: Create the S3 bucket and upload the code

```bash
aws s3api create-bucket --bucket $BUCKET        # us-east-1 needs no LocationConstraint

# Upload everything except the user data and permissions files
aws s3 cp . s3://$BUCKET/ --recursive \
  --exclude "user-data*" --exclude "bucket-permissions*" \
  --exclude "*.zip" --exclude "env.sh" --exclude "README.md"

aws s3 ls s3://$BUCKET/
```

Check that the listing includes every file the user data scripts copy (for example `red-index.html`, `red-root-index.html`, `python.png`).

> Do **not** add a bucket policy that denies `*` to everyone. The instances read the bucket through their IAM role, so no bucket policy is needed.

### Task 2: Create the security group

```bash
export VPC_ID=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true \
  --query 'Vpcs[0].VpcId' --output text)

export SG_ID=$(aws ec2 create-security-group \
  --group-name WebsiteSG --description WebsiteSG \
  --vpc-id $VPC_ID --query GroupId --output text)

aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

echo "export VPC_ID=$VPC_ID SG_ID=$SG_ID" >> env.sh
```

### Task 3: Create the IAM role and launch the Red instance

**3a. IAM policy** (replaces `bucket-permissions.json`):

```bash
cat > s3-arr-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::$BUCKET",
        "arn:aws:s3:::$BUCKET/*"
      ]
    }
  ]
}
EOF

aws iam create-policy --policy-name S3-ARR-Policy \
  --policy-document file://s3-arr-policy.json
```

**3b. Role, instance profile, attach the policy:**

```bash
cat > ec2-trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role --role-name S3-ARR-Role \
  --assume-role-policy-document file://ec2-trust.json

aws iam attach-role-policy --role-name S3-ARR-Role \
  --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/S3-ARR-Policy

# OPTIONAL but recommended: lets you open a shell via Session Manager for debugging
aws iam attach-role-policy --role-name S3-ARR-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# The console creates the instance profile automatically; the CLI needs it explicitly
aws iam create-instance-profile --instance-profile-name S3-ARR-Role
aws iam add-role-to-instance-profile \
  --instance-profile-name S3-ARR-Role --role-name S3-ARR-Role

sleep 15    # IAM propagation; launching too early gives "Invalid IAM Instance Profile"
```

**3c. Look up the AMI and subnets:**

```bash
export AMI_ID=$(aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query Parameter.Value --output text)

export SUBNET_A=$(aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=$VPC_ID Name=availability-zone,Values=us-east-1a \
            Name=default-for-az,Values=true \
  --query 'Subnets[0].SubnetId' --output text)

export SUBNET_B=$(aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=$VPC_ID Name=availability-zone,Values=us-east-1b \
            Name=default-for-az,Values=true \
  --query 'Subnets[0].SubnetId' --output text)

echo "export AMI_ID=$AMI_ID SUBNET_A=$SUBNET_A SUBNET_B=$SUBNET_B" >> env.sh
```

**3d. Prepare the user data** (swap in your bucket name) **and launch Red in us-east-1a:**

```bash
sed "s/YOUR-BUCKET-NAME/$BUCKET/g" user-data-red  > ud-red.sh
sed "s/YOUR-BUCKET-NAME/$BUCKET/g" user-data-blue > ud-blue.sh

grep -c "$BUCKET" ud-red.sh ud-blue.sh      # should be greater than 0 for both

export RED_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID --instance-type t2.micro \
  --subnet-id $SUBNET_A --security-group-ids $SG_ID \
  --associate-public-ip-address \
  --iam-instance-profile Name=S3-ARR-Role \
  --user-data file://ud-red.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Red}]' \
  --query 'Instances[0].InstanceId' --output text)

echo "export RED_ID=$RED_ID" >> env.sh
```

The CLI base64-encodes `file://` user data for you. No key pair is used, matching the lab.

**3e. Check that Red works:**

```bash
aws ec2 wait instance-running --instance-ids $RED_ID

RED_IP=$(aws ec2 describe-instances --instance-ids $RED_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)

sleep 60        # user data (yum update + S3 copy) takes a minute or two
curl -s http://$RED_IP/red/ | head
curl -s http://$RED_IP/ | head
```

### Task 4: Launch the Blue instance in us-east-1b

```bash
export BLUE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID --instance-type t2.micro \
  --subnet-id $SUBNET_B --security-group-ids $SG_ID \
  --associate-public-ip-address \
  --iam-instance-profile Name=S3-ARR-Role \
  --user-data file://ud-blue.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Blue}]' \
  --query 'Instances[0].InstanceId' --output text)

echo "export BLUE_ID=$BLUE_ID" >> env.sh
aws ec2 wait instance-running --instance-ids $BLUE_ID

BLUE_IP=$(aws ec2 describe-instances --instance-ids $BLUE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)

sleep 60
curl -s http://$BLUE_IP/blue/ | head
```

**Checkpoint:** `/red/` works on the Red instance and `/blue/` on the Blue instance.

---

## Exercise 2: Path-based routing

### Task 1: Create the target groups

The health check paths must exist on the instances, so keep them as `/red/index.html` and `/blue/index.html`.

```bash
export RED_TG=$(aws elbv2 create-target-group --name Red \
  --protocol HTTP --port 80 --vpc-id $VPC_ID --target-type instance \
  --health-check-path /red/index.html \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

export BLUE_TG=$(aws elbv2 create-target-group --name Blue \
  --protocol HTTP --port 80 --vpc-id $VPC_ID --target-type instance \
  --health-check-path /blue/index.html \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

aws elbv2 register-targets --target-group-arn $RED_TG  --targets Id=$RED_ID
aws elbv2 register-targets --target-group-arn $BLUE_TG --targets Id=$BLUE_ID

echo "export RED_TG=$RED_TG BLUE_TG=$BLUE_TG" >> env.sh
```

### Task 2: Create the Application Load Balancer

```bash
export ALB_ARN=$(aws elbv2 create-load-balancer --name LabLoadBalancer \
  --type application --scheme internet-facing \
  --subnets $SUBNET_A $SUBNET_B --security-groups $SG_ID \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

aws elbv2 wait load-balancer-available --load-balancer-arns $ALB_ARN

export ALB_DNS=$(aws elbv2 describe-load-balancers --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].DNSName' --output text)

export LISTENER_ARN=$(aws elbv2 create-listener --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$BLUE_TG \
  --query 'Listeners[0].ListenerArn' --output text)

echo "export ALB_ARN=$ALB_ARN ALB_DNS=$ALB_DNS LISTENER_ARN=$LISTENER_ARN" >> env.sh
echo $ALB_DNS
```

The default action sends unmatched requests to **Blue**, as in the lab.

### Task 3: Add the path rules

```bash
aws elbv2 create-rule --listener-arn $LISTENER_ARN --priority 1 \
  --conditions Field=path-pattern,Values='/red*' \
  --actions Type=forward,TargetGroupArn=$RED_TG

aws elbv2 create-rule --listener-arn $LISTENER_ARN --priority 2 \
  --conditions Field=path-pattern,Values='/blue*' \
  --actions Type=forward,TargetGroupArn=$BLUE_TG
```

### Task 4: Wait for healthy targets, then test

```bash
aws elbv2 describe-target-health --target-group-arn $RED_TG \
  --query 'TargetHealthDescriptions[].TargetHealth.State' --output text
aws elbv2 describe-target-health --target-group-arn $BLUE_TG \
  --query 'TargetHealthDescriptions[].TargetHealth.State' --output text
# Repeat until both print: healthy

curl -s http://$ALB_DNS/red/  | head
curl -s http://$ALB_DNS/blue/ | head
```

You can also open `http://<ALB_DNS>/red` and `/blue` in a browser (**console/browser task**).

---

## Exercise 3: Host-based routing

### Task 1: Replace the path rules with host rules

```bash
# Delete the two path rules (keep the default rule)
for ARN in $(aws elbv2 describe-rules --listener-arn $LISTENER_ARN \
  --query 'Rules[?IsDefault==`false`].RuleArn' --output text); do
  aws elbv2 delete-rule --rule-arn $ARN
done

aws elbv2 create-rule --listener-arn $LISTENER_ARN --priority 1 \
  --conditions Field=host-header,Values=red.$DOMAIN \
  --actions Type=forward,TargetGroupArn=$RED_TG

aws elbv2 create-rule --listener-arn $LISTENER_ARN --priority 2 \
  --conditions Field=host-header,Values=blue.$DOMAIN \
  --actions Type=forward,TargetGroupArn=$BLUE_TG
```

**Test before DNS exists** by faking the Host header:

```bash
curl -s -H "Host: red.$DOMAIN"  http://$ALB_DNS/red/  | head
curl -s -H "Host: blue.$DOMAIN" http://$ALB_DNS/blue/ | head
```

### Task 2: Create the Route 53 alias records

```bash
export ZONE_ID=$(aws route53 list-hosted-zones-by-name --dns-name $DOMAIN \
  --query 'HostedZones[0].Id' --output text | sed 's|/hostedzone/||')

export ALB_ZONE=$(aws elbv2 describe-load-balancers --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].CanonicalHostedZoneId' --output text)

echo "export ZONE_ID=$ZONE_ID ALB_ZONE=$ALB_ZONE" >> env.sh

cat > records.json <<EOF
{
  "Changes": [
    { "Action": "UPSERT", "ResourceRecordSet": {
        "Name": "red.$DOMAIN", "Type": "A",
        "AliasTarget": { "HostedZoneId": "$ALB_ZONE", "DNSName": "dualstack.$ALB_DNS", "EvaluateTargetHealth": true } } },
    { "Action": "UPSERT", "ResourceRecordSet": {
        "Name": "blue.$DOMAIN", "Type": "A",
        "AliasTarget": { "HostedZoneId": "$ALB_ZONE", "DNSName": "dualstack.$ALB_DNS", "EvaluateTargetHealth": true } } },
    { "Action": "UPSERT", "ResourceRecordSet": {
        "Name": "$DOMAIN", "Type": "A",
        "AliasTarget": { "HostedZoneId": "$ALB_ZONE", "DNSName": "dualstack.$ALB_DNS", "EvaluateTargetHealth": true } } }
  ]
}
EOF

aws route53 change-resource-record-sets --hosted-zone-id $ZONE_ID \
  --change-batch file://records.json
```

### Task 3: Verify

```bash
dig +short red.$DOMAIN
curl -s http://red.$DOMAIN/red/   | head
curl -s http://blue.$DOMAIN/blue/ | head
curl -s http://$DOMAIN/           | head      # apex goes to the default (Blue) target group
```

Opening `http://red.yourdomain.com` and `http://blue.yourdomain.com` in a browser is the final **browser task**. Use `http://`, since there is no HTTPS listener.

---

## Exercise 4: Clean up

Delete in this order to avoid dependency errors. The ALB and EC2 instances cost money, so don't skip this.

```bash
# 1. Route 53 records (same file, Action changed to DELETE)
sed 's/UPSERT/DELETE/' records.json > delete-records.json
aws route53 change-resource-record-sets --hosted-zone-id $ZONE_ID \
  --change-batch file://delete-records.json

# 2. Load balancer (deletes its listener and rules too), then wait
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN
aws elbv2 wait load-balancers-deleted --load-balancer-arns $ALB_ARN

# 3. Target groups
aws elbv2 delete-target-group --target-group-arn $RED_TG
aws elbv2 delete-target-group --target-group-arn $BLUE_TG

# 4. EC2 instances
aws ec2 terminate-instances --instance-ids $RED_ID $BLUE_ID
aws ec2 wait instance-terminated --instance-ids $RED_ID $BLUE_ID

# 5. Security group (only after the ALB and instances are gone)
aws ec2 delete-security-group --group-id $SG_ID

# 6. IAM
aws iam remove-role-from-instance-profile --instance-profile-name S3-ARR-Role --role-name S3-ARR-Role
aws iam delete-instance-profile --instance-profile-name S3-ARR-Role
aws iam detach-role-policy --role-name S3-ARR-Role --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/S3-ARR-Policy
aws iam detach-role-policy --role-name S3-ARR-Role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam delete-role --role-name S3-ARR-Role
aws iam delete-policy --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/S3-ARR-Policy

# 7. S3 bucket (empty it first)
aws s3 rm s3://$BUCKET --recursive
aws s3api delete-bucket --bucket $BUCKET
```

Confirm nothing is left running:

```bash
aws ec2 describe-instances --filters Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[].InstanceId'
aws elbv2 describe-load-balancers --query 'LoadBalancers[].LoadBalancerName'
```

---

## Debugging and roadblocks

### Quick diagnosis by symptom

| Symptom | Likely cause | Fix |
|---|---|---|
| ALB URL times out | ALB SG blocks port 80, or ALB isn't in public subnets | Check `aws ec2 describe-security-groups --group-ids $SG_ID`; confirm the ALB scheme is `internet-facing` |
| **503 Service Unavailable** | No healthy targets registered | See [target health](#1-check-target-health) |
| **502 Bad Gateway** | Target is up but the app isn't answering on port 80, or the target group port/protocol is wrong | Check httpd on the instance and the target group port (HTTP:80) |
| **404 Not Found** from Apache | Request reached the instance but the path doesn't exist | S3 copy failed in user data; check files under `/var/www/html/red` and `/blue` |
| Target stays `unhealthy` | Health check path doesn't exist on the instance | Health check must be `/red/index.html` (or `/blue/...`) |
| `curl` to `/red/` returns the Blue page | Path rule missing or priority wrong | `aws elbv2 describe-rules --listener-arn $LISTENER_ARN` |
| Host-based test returns the wrong page | Host header doesn't match the rule exactly | Use `curl -H "Host: red.$DOMAIN"`; check the rule values |
| `dig` returns nothing | DNS not propagated or the record is in the wrong hosted zone | Check the `ZONE_ID` and that the zone's NS records match the registrar |
| Browser works for `http` but not `https` | No HTTPS listener | Use `http://`; HTTPS needs an ACM cert and a port 443 listener |

### 1. Check target health

```bash
aws elbv2 describe-target-health --target-group-arn $RED_TG
aws elbv2 describe-target-health --target-group-arn $BLUE_TG
```

Read the `Reason` and `Description` fields:
- `Target.FailedHealthChecks`: the instance answered with a non-200 code or didn't answer. Test the path directly on the instance's public IP.
- `Target.NotRegistered` or `Target.NotInUse`: the instance isn't in the target group or the group isn't attached to a listener rule.
- `Target.Timeout`: the instance SG isn't allowing port 80 from the ALB.

### 2. User data didn't run or failed

Open a shell without SSH keys using Session Manager (the role has `AmazonSSMManagedInstanceCore`; requires the Session Manager plugin for the CLI):

```bash
aws ssm start-session --target $RED_ID
```

Then on the instance:

```bash
sudo tail -n 80 /var/log/cloud-init-output.log     # user data output and errors
systemctl status httpd
ls -R /var/www/html
curl -s localhost/red/
```

No SSM available? Pull the console log from your machine:

```bash
aws ec2 get-console-output --instance-id $RED_ID --latest --output text | tail -n 80
```

Common causes:
- **`YOUR-BUCKET-NAME` was never replaced.** Run `grep YOUR-BUCKET ud-red.sh`; it must return nothing.
- **S3 `AccessDenied`:** the instance profile wasn't attached or the policy ARN doesn't match your bucket. Check `aws ec2 describe-instances --instance-ids $RED_ID --query 'Reservations[0].Instances[0].IamInstanceProfile'`.
- **S3 `NoSuchKey`:** a filename in the user data doesn't exist in the bucket. Compare against `aws s3 ls s3://$BUCKET/`.
- **User data only runs on first boot.** If you edited the script, terminate and relaunch the instance, or run the commands manually on the instance.
- **Still installing:** `yum update -y` can take a couple of minutes. Wait, then retry.

### 3. Fix the IAM instance profile

- **`Invalid IAM Instance Profile name`:** the instance profile wasn't created, or IAM hasn't propagated. Confirm with `aws iam get-instance-profile --instance-profile-name S3-ARR-Role`, wait 15 seconds and retry.
- Fix a bad attachment on a running instance:
  ```bash
  aws ec2 associate-iam-instance-profile --instance-id $RED_ID \
    --iam-instance-profile Name=S3-ARR-Role
  ```
  Then run the S3 copy commands again manually.

### 4. Security group problems

```bash
aws ec2 describe-security-groups --group-ids $SG_ID \
  --query 'SecurityGroups[0].IpPermissions'
```
You should see TCP 80 from `0.0.0.0/0`. If you get `InvalidPermission.Duplicate`, the rule already exists, which is fine.

### 5. Listener rules not behaving

```bash
aws elbv2 describe-rules --listener-arn $LISTENER_ARN \
  --query 'Rules[].{Priority:Priority,Cond:Conditions[0].Values,TG:Actions[0].TargetGroupArn}'
```
- Rules evaluate lowest priority number first; `Default` is last.
- Path patterns are case-sensitive: use `/red*`, not `/Red*`.
- If you left the path rules in place while adding host rules, they can intercept requests. Delete them as shown in Exercise 3.
- A rule's priority can't be reused. Delete the old rule before creating one with the same number (`PriorityInUse`).

### 6. Route 53 and DNS

- **`NoSuchHostedZone` or an empty `ZONE_ID`:** `echo $ZONE_ID`. The domain must have a public hosted zone in this account.
- **`InvalidChangeBatch` (record already exists):** UPSERT is used, so this usually means a conflicting CNAME exists; delete that record first.
- **Alias target rejected:** `ALB_ZONE` must be the ALB's `CanonicalHostedZoneId`, not your own zone's ID.
- **Domain name still not resolving:** DNS can take a few minutes. Meanwhile, test with the `curl -H "Host: ..."` method above or `dig @8.8.8.8 red.$DOMAIN`.
- Check what Route 53 holds:
  ```bash
  aws route53 list-resource-record-sets --hosted-zone-id $ZONE_ID \
    --query 'ResourceRecordSets[?Type==`A`]'
  ```

### 7. Cleanup errors

| Error | Cause | Fix |
|---|---|---|
| `DependencyViolation` deleting the SG | ALB network interfaces or instances still exist | Wait for the ALB deletion and instance termination to finish, then retry |
| `ResourceInUse` deleting a target group | Still attached to a listener or rule | Delete the ALB first (listener and rules go with it) |
| `BucketNotEmpty` | Objects remain | `aws s3 rm s3://$BUCKET --recursive`, then delete |
| `DeleteConflict` deleting the role or policy | Still attached or in an instance profile | Detach policies and remove the role from the profile first |

### 8. General tips

- **Run `aws sts get-caller-identity` and `echo $AWS_DEFAULT_REGION`** when anything says "not found"; resources are regional and the wrong region is a common cause.
- **Empty variables** cause confusing errors. `echo $VAR` before using it, and `source env.sh` in new terminals.
- **Add `--debug`** to any command for the full API request and response.
- **Cost control:** the ALB and instances bill hourly. Run Exercise 4 as soon as you've finished testing.
