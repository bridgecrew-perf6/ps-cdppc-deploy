# Deploy CDP environment

> This step by step tutorial is not intended to be a definitive guide and neither a replacement for the 
> [Cloudera's Oficial Documentation](https://docs.cloudera.com/cdp/latest/index.html). 
>
> Please, use this at your own risk!

## Choose your cloud:

* [AWS](AWS.md)

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

## Set the id bropker mappings
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
    --no-enable-ranger-raz 
```