## AWS Authentication

### Controller's Pod Identity

![Pod Identity Authentication](../pictures/diagrams-provider-aws-auth-pod-identity.png)

Note: If you are using Parameter Store replace `service: SecretsManager` with `service: ParameterStore` in all examples below.

This is basically a zero-configuration authentication method that inherits the credentials from the runtime environment using the [aws sdk default credential chain](https://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/credentials.html#credentials-default).

You can attach a role to the pod using [IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html), [kiam](https://github.com/uswitch/kiam) or [kube2iam](https://github.com/jtblin/kube2iam). When no other authentication method is configured in the `Kind=Secretstore` this role is used to make all API calls against AWS Secrets Manager or SSM Parameter Store.

Based on the Pod's identity you can do a `sts:assumeRole` before fetching the secrets to limit access to certain keys in your provider. This is optional.

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: team-b-store
spec:
  provider:
    aws:
      service: SecretsManager
      region: eu-central-1
      # optional: do a sts:assumeRole before fetching secrets
      role: team-b
```

### Access Key ID & Secret Access Key

![SecretRef](../pictures/diagrams-provider-aws-auth-secret-ref.png)

You can store Access Key ID & Secret Access Key in a `Kind=Secret` and reference it from a SecretStore.

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: team-b-store
spec:
  provider:
    aws:
      service: SecretsManager
      region: eu-central-1
      # optional: assume role before fetching secrets
      role: team-b
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: awssm-secret
            key: access-key
          secretAccessKeySecretRef:
            name: awssm-secret
            key: secret-access-key
```

**NOTE:** In case of a `ClusterSecretStore`, Be sure to provide `namespace` in `accessKeyIDSecretRef`, `secretAccessKeySecretRef` with the namespaces where the secrets reside.

### EKS Service Account credentials

![Service Account](../pictures/diagrams-provider-aws-auth-service-account.png)

This feature lets you use short-lived service account tokens to authenticate with AWS.
You must have [Service Account Volume Projection](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#service-account-token-volume-projection) enabled - it is by default on EKS. See [EKS guide](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts-technical-overview.html) on how to set up IAM roles for service accounts.

The big advantage of this approach is that ESO runs without any credentials.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/team-a
  name: my-serviceaccount
  namespace: default
```

Reference the service account from above in the Secret Store:

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: secretstore-sample
spec:
  provider:
    aws:
      service: SecretsManager
      region: eu-central-1
      auth:
        jwt:
          serviceAccountRef:
            name: my-serviceaccount
```

**NOTE:** In case of a `ClusterSecretStore`, Be sure to provide `namespace` for `serviceAccountRef` with the namespace where the service account resides.

## EKS Pod Identity Setup

In order to use EKS Pod Identity Agent, create a role like this:

```json
{
    "Statement": [
        {
            "Action": [
                "secretsmanager:GetResourcePolicy",
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret",
                "secretsmanager:ListSecretVersionIds"
            ],
            "Effect": "Allow",
            "Resource": [
                "*"
            ]
        }
    ],
    "Version": "2012-10-17"
}
```

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowEksAuthToAssumeRoleForPodIdentity",
            "Effect": "Allow",
            "Principal": {
                "Service": "pods.eks.amazonaws.com"
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession"
            ]
        }
    ]
}

```


Install ESO using helm and define these values:

```yaml
serviceAccount:
  annotations:
  name: external-secrets
```

Create a pod association:

```
aws eks create-pod-identity-association --cluster-name my-cluster --role-arn arn:aws:iam::111122223333:role/my-role --namespace external-secrets --service-account external-secrets
```

Then create a secret store like this:

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: store
spec:
  provider:
    aws:
      service: SecretsManager
      region: eu-central-1
```


_Note_: `serviceAccountRef` _cannot_ be used together with EKS Pod Identity. That's because ESO can not impersonate
service accounts which have iam roles bound using pod identity. Doing so will result in an error like this:
```
unable to create session: an IAM role must be associated with service account ...
```

_Note:_ No `auth` section is defined for the SecretStore.

_Note:_ For even more details you can follow this post for more setup and information using Terraform [here](https://containscloud.com/2024/03/24/integrating-aws-secrets-manager-to-eks-using-external-secrets/).


## Custom Endpoints

You can define custom AWS endpoints if you want to use regional, vpc or custom endpoints. See List of endpoints for [Secrets Manager](https://docs.aws.amazon.com/general/latest/gr/asm.html), [Secure Systems Manager](https://docs.aws.amazon.com/general/latest/gr/ssm.html) and [Security Token Service](https://docs.aws.amazon.com/general/latest/gr/sts.html).

Use the following environment variables to point the controller to your custom endpoints. Note: All resources managed by this controller are affected.

| ENV VAR                     | DESCRIPTION                                                                                                                                                          |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| AWS_SECRETSMANAGER_ENDPOINT | Endpoint for the Secrets Manager Service. The controller uses this endpoint to fetch secrets from AWS Secrets Manager.                                               |
| AWS_SSM_ENDPOINT            | Endpoint for the AWS Secure Systems Manager. The controller uses this endpoint to fetch secrets from SSM Parameter Store.                                            |
| AWS_STS_ENDPOINT            | Endpoint for the Security Token Service. The controller uses this endpoint when creating a session and when doing `assumeRole` or `assumeRoleWithWebIdentity` calls. |
