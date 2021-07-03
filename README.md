# helm-cert-manager

Example Helm chart for setting up Cert Manager in your EKS cluster

## Pre-requisites

### Namespace

Create a new namespace `cert-manager` where we will install the `cert-manager` service.

```bash
kubectl create namespace cert-manager
```

### IAM

We will be using IRSA (IAM Roles for Service Accounts) to give the required permissions to the Cert Manager pod for updating Route53 records for solving DNS challenge.

`Note: You need to create an OIDC provider for your cluster to make use of IRSA. Refer - https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html`

1. Create a new IAM policy `aws-cert-manager-pol` with the policy document at `iam/policy.json`

2. Create a new IAM role `aws-cert-manager-rol` and attach the IAM policy `aws-cert-manager-pol`

3. Update the trust relationship of the IAM role `aws-cert-manager-rol` as below replacing the `account_id`, `eks_cluster_id` and `region` with the appropriate values.

This trust relationship allows pods with serviceaccount `cert-manager` in `cert-manager` namespace to assume the role.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<account_id>:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/<eks_cluster_id>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.<region>.amazonaws.com/id/<eks_cluster_id>:sub": "system:serviceaccount:cert-manager:cert-manager"
        }
      }
    }
  ]
}
```

### Service Account

Create a new service account in the `cert-manager` namespace and associate it with the IAM role which we had created earlier.

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cert-manager
  namespace: cert-manager
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<AWS_ACCOUNT_ID>:role/aws-cert-manager-rol
EOF
```

### Config Updates

In `shared-values.yaml` file available inside `stages` folder, add values for below settings:

|||
|--|--|
|clusterIssuer.email |Email address for getting notified about certificate expiry |

## Install/Upgrade Chart

Run below commands to install/upgrade the cert manager charts.

```bash
helm upgrade -i cert-manager . -n cert-manager --values=stages/shared-values.yaml --values=stages/prod/prod-values.yaml
```
