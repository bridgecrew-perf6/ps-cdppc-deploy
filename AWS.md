# Prepare the AWS Environment

## AWS Region

Check the AWS Region:
```console
aws configure get region
```

Set the region if required:
```console
aws configure set region us-east-1
```

## Pre-reqs

Export the variables to reuse in the next commands:
```console
# Region
export AWS_REGION=$(aws configure get region)

# Tags Helpers
function aws_tags() {
    function aws_tag() {
        echo -n "  {\"Key\": \"$1\", \"Value\": \"$2\"}"
    }
    echo "["
    while [ "$#" -ge 2 ]; do aws_tag "$1" "$2"; echo ","; shift 2; done
    # edit as you need
    aws_tag "tag1" "value1"
    aws_tag "tag2" "value2"
    echo "]"
}

function tag_spec() {
    echo -n "{\"ResourceType\": \"$1\", \"Tags\":"
    shift; aws_tags $*
    echo "}"
}

# CDP Environment
export CDP_ENV_NAME="efranceschi"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity | jq -r .Account)

# Role
export IDBROKER_ROLE=${CDP_ENV_NAME}-idbroker-role

# Storage
export BASE_BUCKET="efranceschi"

# Datalake Storage
export DATALAKE_BUCKET="${BASE_BUCKET}"
export STORAGE_LOCATION_BASE="${DATALAKE_BUCKET}/data"

# Log Storage
export LOGS_BUCKET="${BASE_BUCKET}"
export LOGS_LOCATION_BASE="${LOGS_BUCKET}/logs"

# Backup Storage
export BACKUP_BUCKET="${BASE_BUCKET}"
export BACKUP_LOCATION_BASE="${BACKUP_BUCKET}/backups"

# CDP
export CDP_SERVICE_ACCOUNT_ID="$(cdp environments get-credential-prerequisites --cloud-platform AWS | jq -r .accountId)"
export CDP_EXTERNAL_ID="$(cdp environments get-credential-prerequisites --cloud-platform AWS | jq -r .aws.externalId)"
```

## Create a SSH key pair
Create a local key pair:
```console
ssh-keygen -t rsa -b 2048 -f ${CDP_ENV_NAME} -m PEM
```

Import key-pair to AWS:
```console
aws ec2 import-key-pair \
    --key-name "${CDP_ENV_NAME}-keypair" \
    --public-key-material fileb://${CDP_ENV_NAME}.pub \
    --tag-specifications "$(tag_spec key-pair)"
```

## Create the VPC
```console
# Example: /21 network = 246 hosts with 8 x /24 subnets
# See: http://jodies.de/ipcalc?host=10.10.0.0&mask1=21&mask2=25

# VPC 10.10.0.0/21
export VPC_ID=$(aws ec2 create-vpc \
    --cidr-block 10.10.0.0/21 \
    --tag-specifications "$(tag_spec vpc Name ${CDP_ENV_NAME}-vpc)" \
    | jq -r .Vpc.VpcId
)
```

## Create an Internet Gateway
```console
export IGW_ID=$(aws ec2 create-internet-gateway \
    --tag-specifications "$(tag_spec internet-gateway Name ${CDP_ENV_NAME}-igw)" \
    | jq -r .InternetGateway.InternetGatewayId
)

aws ec2 attach-internet-gateway \
    --internet-gateway-id "${IGW_ID}" \
    --vpc-id "${VPC_ID}"
```

## Create Private Subnets
```console
# Private Subnet 10.10.0.0/25 AZ-A
export PRIV_SUB1A=$(aws ec2 create-subnet \
    --vpc-id "${VPC_ID}" \
    --cidr-block 10.10.0.0/25 \
    --availability-zone "${AWS_REGION}a" \
    --tag-specifications "$(tag_spec subnet Name ${CDP_ENV_NAME}-priv-subnet-1a)" \
    | jq -r .Subnet.SubnetId
)

# Private Subnet 10.10.0.128/25 AZ-B
export PRIV_SUB1B=$(aws ec2 create-subnet \
    --vpc-id "${VPC_ID}" \
    --cidr-block 10.10.0.128/25 \
    --availability-zone "${AWS_REGION}b" \
    --tag-specifications "$(tag_spec subnet Name ${CDP_ENV_NAME}-priv-subnet-1b)" \
    | jq -r .Subnet.SubnetId
)

# Private Subnet 10.10.1.0/25 AZ-A
export PRIV_SUB2A=$(aws ec2 create-subnet \
    --vpc-id "${VPC_ID}" \
    --cidr-block 10.10.1.0/25 \
    --availability-zone "${AWS_REGION}a" \
    --tag-specifications "$(tag_spec subnet Name ${CDP_ENV_NAME}-priv-subnet-2a)" \
    | jq -r .Subnet.SubnetId
)

# Private Subnet 10.10.1.0/25 AZ-B
export PRIV_SUB2B=$(aws ec2 create-subnet \
    --vpc-id "${VPC_ID}" \
    --cidr-block 10.10.1.128/25 \
    --availability-zone "${AWS_REGION}b" \
    --tag-specifications "$(tag_spec subnet Name ${CDP_ENV_NAME}-priv-subnet-2b)" \
    | jq -r .Subnet.SubnetId
)
```

## Create Public Subnets
```console
# Public Subnet 10.10.6.0/25 AZ-A
export PUB_SUB1A=$(aws ec2 create-subnet \
    --vpc-id "${VPC_ID}" \
    --cidr-block 10.10.6.0/25 \
    --availability-zone "${AWS_REGION}a" \
    --tag-specifications "$(tag_spec subnet Name ${CDP_ENV_NAME}-pub-subnet-1a)" \
    | jq -r .Subnet.SubnetId
)

# Public Subnet 10.10.6.0/25 AZ-B
export PUB_SUB1B=$(aws ec2 create-subnet \
    --vpc-id "${VPC_ID}" \
    --cidr-block 10.10.6.128/25 \
    --availability-zone "${AWS_REGION}b" \
    --tag-specifications "$(tag_spec subnet Name ${CDP_ENV_NAME}-pub-subnet-1b)" \
    | jq -r .Subnet.SubnetId
)
```

## Allocate Addresses
```console
EIP_A=$(aws ec2 allocate-address --domain vpc | jq -r .AllocationId)
EIP_B=$(aws ec2 allocate-address --domain vpc | jq -r .AllocationId)
```

## Create a NAT Gateway
```console
# AZ-A
export NGW_A=$(aws ec2 create-nat-gateway \
    --subnet-id "${PUB_SUB1A}" \
    --allocation-id "${EIP_A}" \
    --tag-specifications "$(tag_spec natgateway Name ${CDP_ENV_NAME}-nat-a)" \
    | jq -r .NatGateway.NatGatewayId
)

# AZ-B
export NGW_B=$(aws ec2 create-nat-gateway \
    --subnet-id "${PUB_SUB1B}" \
    --allocation-id "${EIP_B}" \
    --tag-specifications "$(tag_spec natgateway Name ${CDP_ENV_NAME}-nat-b)" \
    | jq -r .NatGateway.NatGatewayId
)
```

## Create Routing Table for Private Subnets
```console
# AZ-A
export PRIV_RTB_A_ID=$(aws ec2 create-route-table \
    --vpc-id "${VPC_ID}" \
    --tag-specifications "$(tag_spec route-table Name ${CDP_ENV_NAME}-igw)" \
    | jq -r .RouteTable.RouteTableId
)

aws ec2 create-route \
    --route-table-id "${PRIV_RTB_A_ID}" \
    --destination-cidr-block "0.0.0.0/0" \
    --nat-gateway-id "${NGW_A}"

# AZ-B
export PRIV_RTB_B_ID=$(aws ec2 create-route-table \
    --vpc-id "${VPC_ID}" \
    --tag-specifications "$(tag_spec route-table Name ${CDP_ENV_NAME}-igw)" \
    | jq -r .RouteTable.RouteTableId
)

aws ec2 create-route \
    --route-table-id "${PRIV_RTB_B_ID}" \
    --destination-cidr-block "0.0.0.0/0" \
    --nat-gateway-id "${NGW_B}"
```

## Create Routing Table for Public Subnets
```console
export PUB_RTB_ID=$(aws ec2 create-route-table \
    --vpc-id "${VPC_ID}" \
    --tag-specifications "$(tag_spec route-table Name ${CDP_ENV_NAME}-igw)" \
    | jq -r .RouteTable.RouteTableId
)

aws ec2 create-route \
    --route-table-id "${PUB_RTB_ID}" \
    --destination-cidr-block "0.0.0.0/0" \
    --nat-gateway-id "${IGW_ID}"
```

# Enable map public ip on launch
```console
aws ec2 modify-subnet-attribute \
    --subnet-id "${PUB_SUB1A}" \
    --map-public-ip-on-launch

aws ec2 modify-subnet-attribute \
    --subnet-id "${PUB_SUB1B}" \
    --map-public-ip-on-launch
```

# Replace the routing table
```console
aws ec2 associate-route-table \
    --route-table-id "${PUB_RTB_ID}" \
    --subnet-id "${PUB_SUB1A}"

aws ec2 associate-route-table \
    --route-table-id "${PUB_RTB_ID}" \
    --subnet-id "${PUB_SUB1B}"

aws ec2 associate-route-table \
    --route-table-id "${PRIV_RTB_A_ID}" \
    --subnet-id "${PRIV_SUB1A}"

aws ec2 associate-route-table \
    --route-table-id "${PRIV_RTB_A_ID}" \
    --subnet-id "${PRIV_SUB2A}"

aws ec2 associate-route-table \
    --route-table-id "${PRIV_RTB_B_ID}" \
    --subnet-id "${PRIV_SUB1B}"

aws ec2 associate-route-table \
    --route-table-id "${PRIV_RTB_B_ID}" \
    --subnet-id "${PRIV_SUB2B}"

```

## Create a S3 Buckets
```console
# Datalake
aws s3api create-bucket --bucket ${DATALAKE_BUCKET}

# Logs
aws s3api create-bucket --bucket ${LOGS_BUCKET}

# Backup
aws s3api create-bucket --bucket ${BACKUP_BUCKET}
```

## Create Policies

Use the following to create a default policy:
```console
aws iam create-policy \
    --policy-name "${CDP_ENV_NAME}-full-policy" \
    --description "CDP Policy for the environment ${CDP_ENV_NAME}" \
    --tags "$(aws_tags)" \
    --policy-document "$(envsubst < resources/aws/default-policy.json)"
```

OR 

Use the following to create a more restricted policy:
```console
aws iam create-policy \
    --policy-name "${CDP_ENV_NAME}-policy" \
    --description "CDP Policy for the environment ${CDP_ENV_NAME}" \
    --tags "$(aws_tags)" \
    --policy-document "$(envsubst < resources/aws/minimal-policy.json)"

aws iam create-policy \
    --policy-name "${CDP_ENV_NAME}-log-policy" \
    --description "CDP Log Policy for the environment ${CDP_ENV_NAME}" \
    --tags "$(aws_tags)" \
    --policy-document "$(envsubst < resources/aws/aws-cdp-log-policy.json)"

aws iam create-policy \
    --policy-name "${CDP_ENV_NAME}-backup-policy" \
    --description "CDP Backup Policy for the environment ${CDP_ENV_NAME}" \
    --tags "$(aws_tags)" \
    --policy-document "$(envsubst < resources/aws/aws-cdp-backup-policy.json)"
    
aws iam create-policy \
    --policy-name "${CDP_ENV_NAME}-ranger-audit-policy" \
    --description "CDP Ranger Audit Policy for the environment ${CDP_ENV_NAME}" \
    --tags "$(aws_tags)" \
    --policy-document "$(envsubst < resources/aws/aws-cdp-ranger-audit-s3-policy.json)"

aws iam create-policy \
    --policy-name "${CDP_ENV_NAME}-datalake-admin-policy" \
    --description "CDP Datalake Admin Policy for the environment ${CDP_ENV_NAME}" \
    --tags "$(aws_tags)" \
    --policy-document "$(envsubst < resources/aws/aws-cdp-datalake-admin-s3-policy.json)"

aws iam create-policy \
    --policy-name "${CDP_ENV_NAME}-bucket-access-policy" \
    --description "CDP Bucket Access Policy for the environment ${CDP_ENV_NAME}" \
    --tags "$(aws_tags)" \
    --policy-document "$(envsubst < resources/aws/aws-cdp-bucket-access-policy.json)"

aws iam create-policy \
    --policy-name "${CDP_ENV_NAME}-datalake-backup-policy" \
    --description "CDP Datalake Backup Policy for the environment ${CDP_ENV_NAME}" \
    --tags "$(aws_tags)" \
    --policy-document "$(envsubst < resources/aws/aws-cdp-backup-policy.json)"

aws iam create-policy \
    --policy-name "${CDP_ENV_NAME}-idbroker-assume-role-policy" \
    --description "CDP IdBroker Assume Role Policy for the environment ${CDP_ENV_NAME}" \
    --tags "$(aws_tags)" \
    --policy-document "$(envsubst < resources/aws/aws-cdp-idbroker-assume-role-policy.json)"
```

## Create Roles
Use the following to create roles:

```console
# IDBROKER_ROLE
aws iam create-role \
    --role-name "${CDP_ENV_NAME}-idbroker-role" \
    --description "CDP IDBROKER Role for the environment ${CDP_ENV_NAME}" \
    --tags "$(aws_tags)" \
    --path "/" \
    --assume-role-policy-document "$(envsubst < resources/aws/aws-cdp-ec2-role-trust-policy.json)"

aws iam attach-role-policy \
    --role-name "${CDP_ENV_NAME}-idbroker-role" \
    --policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/${CDP_ENV_NAME}-idbroker-assume-role-policy"

aws iam attach-role-policy \
    --role-name "${CDP_ENV_NAME}-idbroker-role" \
    --policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/${CDP_ENV_NAME}-log-policy"

# LOG_ROLE
aws iam create-role \
    --role-name "${CDP_ENV_NAME}-log-role" \
    --description "CDP LOG Role for the environment ${CDP_ENV_NAME}" \
    --tags "$(aws_tags)" \
    --path "/" \
    --assume-role-policy-document "$(envsubst < resources/aws/aws-cdp-ec2-role-trust-policy.json)"

aws iam attach-role-policy \
    --role-name "${CDP_ENV_NAME}-log-role" \
    --policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/${CDP_ENV_NAME}-log-policy"

aws iam attach-role-policy \
    --role-name "${CDP_ENV_NAME}-log-role" \
    --policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/${CDP_ENV_NAME}-backup-policy"

# RANGER_AUDIT_ROLE
aws iam create-role \
    --role-name "${CDP_ENV_NAME}-ranger-audit-role" \
    --description "CDP RANGER_AUDIT Role for the environment ${CDP_ENV_NAME}" \
    --tags "$(aws_tags)" \
    --path "/" \
    --assume-role-policy-document "$(envsubst < resources/aws/aws-cdp-idbroker-role-trust-policy.json)"

aws iam attach-role-policy \
    --role-name "${CDP_ENV_NAME}-ranger-audit-role" \
    --policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/${CDP_ENV_NAME}-ranger-audit-policy"

aws iam attach-role-policy \
    --role-name "${CDP_ENV_NAME}-ranger-audit-role" \
    --policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/${CDP_ENV_NAME}-bucket-access-policy"

aws iam attach-role-policy \
    --role-name "${CDP_ENV_NAME}-ranger-audit-role" \
    --policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/${CDP_ENV_NAME}-datalake-backup-policy"

# DATALAKE_ADMIN_ROLE
aws iam create-role \
    --role-name "${CDP_ENV_NAME}-datalake-admin-role" \
    --description "CDP DATALAKE_ADMIN Role for the environment ${CDP_ENV_NAME}" \
    --tags "$(aws_tags)" \
    --path "/" \
    --assume-role-policy-document "$(envsubst < resources/aws/aws-cdp-idbroker-role-trust-policy.json)"

aws iam attach-role-policy \
    --role-name "${CDP_ENV_NAME}-datalake-admin-role" \
    --policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/${CDP_ENV_NAME}-datalake-admin-policy"

aws iam attach-role-policy \
    --role-name "${CDP_ENV_NAME}-datalake-admin-role" \
    --policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/${CDP_ENV_NAME}-bucket-access-policy"

aws iam attach-role-policy \
    --role-name "${CDP_ENV_NAME}-datalake-admin-role" \
    --policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/${CDP_ENV_NAME}-datalake-backup-policy"
```

## Instance Profiles

Use the following to create instance profiles:

```console
# DATA ACCESS INSTANCE PROFILE
aws iam create-instance-profile \
    --instance-profile-name "${CDP_ENV_NAME}-data-access-instance-profile" \
    --tags "$(aws_tags)"

aws iam add-role-to-instance-profile \
    --instance-profile-name "${CDP_ENV_NAME}-data-access-instance-profile" \
    --role-name "${CDP_ENV_NAME}-idbroker-role"

# LOG ACCESS INSTANCE PROFILE
aws iam create-instance-profile \
    --instance-profile-name "${CDP_ENV_NAME}-log-access-instance-profile" \
    --tags "$(aws_tags)"

aws iam add-role-to-instance-profile \
    --instance-profile-name "${CDP_ENV_NAME}-log-access-instance-profile" \
    --role-name "${CDP_ENV_NAME}-idbroker-role"
```

## Cross Account Role

Use the following to create the cross account role:

```console
aws iam create-role \
    --role-name "${CDP_ENV_NAME}-cross-account-role" \
    --description "CDP Cross Account Role for the environment ${CDP_ENV_NAME}" \
    --tags "$(aws_tags)" \
    --path "/" \
    --assume-role-policy-document "$(envsubst < resources/aws/aws-cdp-cross-account-assume-role.json)"
```

Decide the correct assignment:

Full:
```console
aws iam attach-role-policy \
    --role-name "${CDP_ENV_NAME}-cross-account-role" \
    --policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/${CDP_ENV_NAME}-full-policy"
```

OR Minimal:

```console
aws iam attach-role-policy \
    --role-name "${CDP_ENV_NAME}-cross-account-role" \
    --policy-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:policy/${CDP_ENV_NAME}-policy"
```

# Deploy CDP

## Create a CDP Credential
Use the following to create the CDP Credential:

```console
cdp environments create-aws-credential \
    --credential-name "${CDP_ENV_NAME}" \
    --role-arn "arn:aws:iam::${AWS_ACCOUNT_ID}:role/${CDP_ENV_NAME}-cross-account-role" \
    --description "${CDP_ENV_NAME} @ AWS" 
```

## Create the CDP Environment
```console
cdp environments create-aws-environment \
    --environment-name "${CDP_ENV_NAME}" \
    --credential-name "${CDP_ENV_NAME}" \
    --region "${AWS_REGION}" \
    --security-access cidr=0.0.0.0/0 \
    --description "${CDP_ENV_NAME} Environment" \
    --endpoint-access-gateway-scheme PUBLIC \
    --enable-tunnel \
    --authentication publicKeyId="${CDP_ENV_NAME}-keypair" \
    --log-storage storageLocationBase=s3a://${LOGS_LOCATION_BASE},instanceProfile=arn:aws:iam::${AWS_ACCOUNT_ID}:instance-profile/${CDP_ENV_NAME}-data-access-instance-profile \
    --vpc-id ${VPC_ID} \
    --subnet-ids ${PRIV_SUB1A} ${PRIV_SUB1B} ${PRIV_SUB2A} ${PRIV_SUB2B} ${PUB_SUB1A} ${PUB_SUB1B} \
    --no-create-service-endpoints \
    --no-create-private-subnets \
    --free-ipa instanceCountByGroup=2 
```

## Set the id broker mappings
```console
cdp environments set-id-broker-mappings \
    --environment-name "${CDP_ENV_NAME}" \
    --data-access-role arn:aws:iam::${AWS_ACCOUNT_ID}:role/${CDP_ENV_NAME}-datalake-admin-role \
    --ranger-audit-role arn:aws:iam::${AWS_ACCOUNT_ID}:role/${CDP_ENV_NAME}-ranger-audit-role \
    --set-empty-mappings 
```

## Create the CDP Datalake
```console
cdp datalake create-aws-datalake \
    --datalake-name "${CDP_ENV_NAME}-datalake" \
    --environment-name "${CDP_ENV_NAME}" \
    --cloud-provider-configuration instanceProfile=arn:aws:iam::${AWS_ACCOUNT_ID}:instance-profile/${CDP_ENV_NAME}-data-access-instance-profile,storageBucketLocation=s3a://${STORAGE_LOCATION_BASE} \
    --scale LIGHT_DUTY \
    --runtime 7.2.12 \
    --enable-ranger-raz 
```
