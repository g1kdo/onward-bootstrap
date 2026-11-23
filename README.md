# onward-bootstrap – Foundation for the delivery photos service

Everything that has to exist **before** the photos infrastructure and application can deploy: the bucket that holds nested templates, the image registry, the GitHub OIDC roles, the role CloudFormation uses for the photos stack, and the second-Region replica bucket.

| Repository | Responsibility |
|---|---|
| **onward-bootstrap** (this one) | Foundation, deployed once and rarely changed |
| [onward-repo-infra](https://github.com/g1kdo/onward-repo-infra) `photos/` | Photos platform: network, database, storage, load balancer, CDN, ECS service |
| [onward-photos-app](https://github.com/g1kdo/onward-photos-app) | Java application, Dockerfile, release workflow |

## Stacks

| Template | Stack name | Region | Creates |
|---|---|---|---|
| `stacks/github-access.yaml` | `onward-dev-bootstrap-github-access-stack` | eu-west-1 | Templates bucket, ECR repo `onward-dev-photos-app`, OIDC roles for the infra and app repos |
| `stacks/photos-stack-role.yaml` | `onward-dev-bootstrap-photos-stack-role-stack` | eu-west-1 | `service-role/onward-dev-photos-stack-role`, used by GitSync for the photos root stack |
| `stacks/photo-replica.yaml` | `onward-dev-bootstrap-photo-replica-stack` | **eu-central-1** | Versioned replica bucket that receives every photo |

All three are deployed by GitSync from their `*-deployment.yaml` file, using the one role created by hand.

## Manual steps (once, in CloudShell)

**1. Bootstrap stack role** (the only role not in a template; replace `<ACCOUNT_ID>` in the policy first):

```bash
aws iam create-role --role-name onward-dev-bootstrap-stack-role --path /service-role/ \
  --assume-role-policy-document file://manual/bootstrap-stack-trust-policy.json
aws iam create-policy --policy-name onward-dev-bootstrap-stack-policy \
  --policy-document file://manual/bootstrap-stack-policy.json
aws iam attach-role-policy --role-name onward-dev-bootstrap-stack-role \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/onward-dev-bootstrap-stack-policy
```

**2. CloudFront signing key** (the private key never leaves AWS or your CloudShell session):

```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out private_key.pem
openssl pkey -pubout -in private_key.pem -out public_key.pem
aws secretsmanager create-secret --name onward-dev-photos-cloudfront-signing-key \
  --secret-string file://private_key.pem \
  --tags Key=Project,Value=onward Key=Owner,Value="Katy Great Adonaï" Key=Environment,Value=dev
cat public_key.pem      # goes into photos/root-deployment.yaml in the infra repo
rm private_key.pem
```

**3. GitHub settings** (values come from the `github-access` stack outputs):

| Repository | Kind | Name | Value |
|---|---|---|---|
| onward-repo-infra | Secret | `PHOTOS_TEMPLATES_ROLE_ARN` | `InfraPublisherRoleArn` |
| onward-repo-infra | Variable | `PHOTOS_TEMPLATES_BUCKET` | `TemplatesBucketName` |
| onward-photos-app | Secret | `AWS_ROLE_ARN` | `AppReleaseRoleArn` |
| onward-photos-app | Variables | `AWS_REGION`, `ECR_REPOSITORY`, `ECS_CLUSTER`, `ECS_SERVICE` | `eu-west-1`, `onward-dev-photos-app`, `onward-dev-photos-cluster`, `onward-dev-photos-service` |

## Notes

- GitHub issues immutable OIDC subjects (`repo:<owner>@<id>/<repo>@<id>`); read them with `gh api repos/<owner>/<repo>/actions/oidc/customization/sub`.
- The photos stack role lives under `/service-role/`, so it cannot edit itself through its own `onward-dev-photos-*` role permissions.
- If GitSync is not available for the replica stack in eu-central-1, deploy it from CloudShell:
  `aws cloudformation deploy --region eu-central-1 --stack-name onward-dev-bootstrap-photo-replica-stack --template-file stacks/photo-replica.yaml --tags Project=onward Owner="Katy Great Adonaï" Environment=dev`
