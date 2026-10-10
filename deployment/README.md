# VecStream Lambda deployment

Deploys L1, L2, and Kafka search Lambdas into a VPC with Gateway endpoints
for standard S3 and S3 Express One Zone. 

This README covers the Lambda fleet at the root of `deployment/`. Two sibling
subfolders hold the rest of the deployment stacks:

- [kafka_on_eks/](kafka_on_eks/README.md): Apache Kafka on EKS with the
  Strimzi operator, including optional tiered storage to S3.
- [benchmark_clients/](benchmark_clients/README.md): the EC2 benchmark client
  for the ingestion, static queries and streaming queries suites, one
  terraform workspace per suite plus the ansible playbooks that provision it.

## Prerequisites

- AWS CLI v2, authenticated against the target account
  (`aws sts get-caller-identity` should succeed)
- Terraform >= 1.5
- `uv` (for the layer build)
- By default, deployment auto-attaches Kafka search Lambdas to the Kafka-on-EKS
  VPC/subnets by reading `kafka_on_eks/terraform/terraform.tfstate` (when
  present). If that state file is missing, Kafka Lambdas stay outside any VPC.
  L1/L2 Lambdas keep the historical default (no VPC). You can always override
  network settings via `-var` flags; see "Customizing the VPC" below.

## Build the deployment artifacts

```bash
cd deployment

# 1. Package the vecstream/ code as code.zip
bash package_lambda.sh

# 2. Build the Lambda layer (numpy, faiss-cpu, aiohttp, confluent-kafka, mmh3)
bash package_layer.sh

# 3. Publish the layer; note the new Version number in the output
bash publish-layer.sh
```

`publish-layer.sh` runs `aws lambda publish-layer-version` and prints the new
`Version` (e.g. `3`). The aiohttp layer is only published the first time; if
you need to re-publish it, see the script for the exact command.

## Pin the new layer version

Open `variables.tf` and update the default for `lambda_layers` to the version
returned by `publish-layer.sh`. Example after bumping `vecstream` to v3:

```hcl
variable "lambda_layers" {
  default = [
    "arn:aws:lambda:us-east-1:012345678910:layer:vecstream:3",
    "arn:aws:lambda:us-east-1:012345678910:layer:aiohttp:1",
  ]
}
```

## Deploy

```bash
cd deployment

terraform init

# Optional: review the plan first
terraform plan -out=tfplan

terraform apply tfplan
```

By default this creates 500 L1, 500 L2, and 25 Kafka lambdas. When
`kafka_on_eks/terraform/terraform.tfstate` exists, Kafka lambdas are attached
to that VPC/private subnets. L1/L2 remain outside any VPC unless explicitly
configured. Approximate apply time: a few minutes (Lambda creation is parallel;
for VPC-attached Kafka lambdas the first apply also creates ENIs in the VPC,
which AWS rate-limits per account).

## Wire the URLs into the client

```bash
cd deployment

terraform output -json \
  | jq '{l1: .l1_lambda_function_urls.value, l2: .l2_lambda_function_urls.value, kafka: .kafka_lambda_function_urls.value}'
```

Copy the lists into `vecstream/urls.py`:

- `l1_lambda_function_urls` -> `l1_cache_lambda_urls`
- `l2_lambda_function_urls` -> `l2_cache_lambda_urls`
- `kafka_lambda_function_urls` -> `kafka_search_lambda_urls`

Redeploy the lambdas, to update the `urls.py` in each function. 

## Smoke test

Each tier responds to the `WARMUP` query type with a 200:

```bash
L1_URL=$(terraform output -json l1_lambda_function_urls | jq -r '.[0]')
L2_URL=$(terraform output -json l2_lambda_function_urls | jq -r '.[0]')
KAFKA_URL=$(terraform output -json kafka_lambda_function_urls | jq -r '.[0]')

curl -sS -X POST "$L1_URL" \
  -H 'content-type: application/json' \
  -d '{"query_type": "WARMUP"}' | jq

curl -sS -X POST "$L2_URL" \
  -H 'content-type: application/json' \
  -d '{"query_type": "WARMUP"}' | jq

curl -sS -X POST "$KAFKA_URL" \
  -H 'content-type: application/json' \
  -d '{"query_type": "WARMUP"}' | jq
```

All three should return `{"message": "Lambda warmup complete."}` (L1/L2) or
`{"message": "Warmup completed successfully."}` (Kafka) with HTTP 200.

## Customizing the VPC

By default, if the Kafka-on-EKS terraform state file is available, Kafka
lambdas are attached to that stack's VPC/private subnets. Setting `vpc_id`
explicitly overrides auto-discovery for Kafka (and async index) lambdas.

```bash
terraform apply \
  -var 'vpc_id=vpc-0123456789abcdef0' \
  -var 'subnet_ids=["subnet-aaa","subnet-bbb"]' \
  -var 'security_group_ids=["sg-0123456789abcdef0"]' \
  -var 's3_gateway_route_table_ids=["rtb-aaa","rtb-bbb"]' \
  -var 's3express_gateway_route_table_ids=["rtb-aaa"]'
```

If `subnet_ids` is omitted, deployment prefers Kafka-on-EKS private subnets
when auto-discovered, otherwise all subnets in the resolved `vpc_id`. If
`security_group_ids` is omitted, deployment creates a dedicated
`vecstream-lambda-sg` and uses it. Custom subnets must all be in the same VPC
as `vpc_id`.

## Customizing lambda counts

```bash
terraform apply \
  -var 'lambda_count_l1=200' \
  -var 'lambda_count_l2=200' \
  -var 'lambda_count_kafka=10'
```

## Tear down

```bash
cd deployment
terraform destroy
```

This removes all lambdas, function URLs, the IAM role, the S3 code bucket
(forced), and the VPC endpoints. The Lambda code package on S3 is gone with
the bucket. The published layer in AWS Lambda is NOT removed by Terraform
(destroy it manually with `aws lambda delete-layer-version` if you want to
clean it up).
