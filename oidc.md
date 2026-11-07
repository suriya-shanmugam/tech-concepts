# Understanding OIDC and Implementing IRSA in AWS EKS: A Step-by-Step Guide

## Introduction

When deploying applications on Amazon EKS, we often need our Kubernetes pods to communicate with other AWS services such as:

- Amazon S3
- Amazon DynamoDB
- Amazon SQS
- AWS Secrets Manager
- Amazon RDS

But how does a Kubernetes pod authenticate itself with AWS?

One common approach is to provide AWS access keys to the application. However, storing and managing static AWS credentials inside Kubernetes pods is not a good security practice.

AWS provides a better solution called **IRSA (IAM Roles for Service Accounts)**.

IRSA uses OIDC (OpenID Connect) and AWS STS (Security Token Service) to provide temporary AWS credentials to Kubernetes workloads.

Before understanding IRSA, let's understand OIDC with a simple real-world example.

---

# Part 1: Understanding OIDC with a Simple Example

## 1\. What is OIDC?

OIDC stands for **OpenID Connect**.

It is an authentication protocol that allows one application to verify a user's identity through another trusted service.

In simple terms:

OIDC allows you to log in to one application using your account from another application.

For example:

Imagine you visit an online shopping website called `shop.com`.

Instead of creating a new username and password, you click:

**Continue with Google**

What happens behind the scenes?

### Step 1: User clicks Login with Google

You visit Shop.com and click the Login with Google button.

Shop.com redirects you to Google's authentication page.

### Step 2: Google authenticates the user

Google verifies your identity by checking your credentials and, if required, your MFA authentication.

### Step 3: Google generates an ID token

After successful authentication, Google issues an ID token containing information about your identity.

A simplified example:

json

```json
{
  "iss": "https://accounts.google.com",
  "sub": "123456789",
  "email": "user@gmail.com",
  "email_verified": true,
  "aud": "shop.com-client-id"
}
```

Let's understand these fields:

| Field | Meaning |
| --- | --- |
| iss | Issuer – who generated the token |
| sub | Subject – unique user identifier |
| email | User's email address |
| aud | Audience – application for which the token is intended |

### Step 4: Shop.com validates the token

Shop.com verifies:

- Was this token issued by Google?
- Is the token signature valid?
- Has the token expired?
- Is this token intended for Shop.com?

If the token is valid, Shop.com can trust the authenticated identity.

### Step 5: User successfully logs in

Shop.com creates an application session, and you can start shopping.

### OIDC Architecture

text

```text
       USER
         |
         | Click Login with Google
         |
         v
   SHOP.COM
         |
         | Redirect to Google
         |
         v
      GOOGLE
   (Identity Provider)
         |
         | Authenticate user
         |
         | Generate ID Token
         |
         v
   SHOP.COM
         |
         | Validate ID Token
         |
         v
   LOGIN SUCCESSFUL
```

**The key takeaway:**

Google acts as the Identity Provider (IdP), and Shop.com trusts Google to authenticate users.

This same fundamental concept is used in AWS EKS, but instead of authenticating a human user, we authenticate a Kubernetes workload.

---

# Part 2: Understanding OIDC in AWS EKS

## 2\. Why do we need OIDC in EKS?

Imagine we have an application running inside an EKS cluster.

Our application needs to download files from an S3 bucket.

For example:

text

```text
EKS Cluster
   |
   |-- Application Pod
           |
           |-- Needs access to S3
```

The application needs AWS permissions to access S3.

How can AWS identify this pod and decide whether it should be allowed to access S3?

This is where IRSA comes into the picture.

IRSA allows us to associate an AWS IAM role with a Kubernetes ServiceAccount.

The pod uses that ServiceAccount, and AWS can verify the workload's identity using OIDC.

After successful verification, AWS STS provides temporary AWS credentials.

The application uses these credentials to communicate with S3.

## 3\. Important components involved in IRSA

Before implementing IRSA, let's understand the responsibility of each component.

### A. EKS OIDC Issuer

Every Amazon EKS cluster has a unique OIDC issuer URL.

Example:

text

```text
https://oidc.eks.us-east-1.amazonaws.com/id/EXAMPLE123
```

EKS acts as the issuer of projected Kubernetes ServiceAccount tokens.

These tokens contain information about the Kubernetes workload, including:

- The cluster's OIDC issuer (`iss`).
- The ServiceAccount identity (`sub`).
- The intended audience (`aud`).

For example:

text

```text
sub = system:serviceaccount:default:s3-reader

aud = sts.amazonaws.com
```

This tells us that the token represents the `s3-reader` ServiceAccount in the `default` namespace and is intended for AWS STS.

### B. IAM OIDC Identity Provider

AWS IAM needs to trust the OIDC issuer before it can accept tokens issued by that EKS cluster.

Therefore, we register the EKS OIDC issuer URL as an identity provider in AWS IAM.

This establishes trust between the EKS cluster's OIDC issuer and AWS IAM.

In simple words:

We are telling AWS IAM:

"I trust identity tokens issued by this EKS cluster, provided they pass token validation."

AWS uses the provider's discovery information and public signing keys to validate tokens.

**Important:** Creating an IAM OIDC provider does not automatically give every pod AWS permissions. It only establishes a trusted identity provider.

### C. IAM Role

An IAM role defines what an application is allowed to do in AWS.

For example, we can create an IAM role with permission to read objects from a particular S3 bucket.

The role has two important configurations:

1. Trust policy – who is allowed to assume this role?
2. Permissions policy – what can the role do after it is assumed?

In IRSA, the trust policy identifies the trusted EKS OIDC provider and restricts access to a particular Kubernetes ServiceAccount.

### D. AWS STS (Security Token Service)

STS stands for Security Token Service.

It is an AWS service that provides temporary security credentials.

In IRSA, the pod presents its projected OIDC token to AWS STS using:

text

```text
AssumeRoleWithWebIdentity
```

STS validates the token and checks the IAM role's trust policy.

If everything is valid, STS returns temporary AWS credentials:

- Access Key ID
- Secret Access Key
- Session Token

The application can then use these temporary credentials to access AWS services.

---

# Part 3: How EKS, OIDC, IAM and STS Work Together

## 4\. Complete IRSA authentication flow

Let's connect all the components.

Imagine the following setup:

- EKS cluster: `my-cluster`
- Kubernetes namespace: `default`
- ServiceAccount: `s3-reader`
- IAM role: `s3-read-role`
- S3 bucket: `my-app-data-bucket`

The authentication flow looks like this:

text

```text
                 EKS CLUSTER
                      |
                      |
              APPLICATION POD
                      |
                      |
              ServiceAccount
                 s3-reader
                      |
                      |
          Projected OIDC Token
                      |
                      |
                      v
                 AWS STS
        AssumeRoleWithWebIdentity
                      |
                      |
             Validate OIDC Token
                      |
                      |
                      v
                AWS IAM ROLE
                 s3-read-role
                      |
                      |
              Trust Policy Check
                      |
                      |
             Permissions Policy
                      |
                      |
                      v
             TEMPORARY AWS
               CREDENTIALS
                      |
                      |
                      v
                  AMAZON S3
```

Let's understand each step.

**Step 1:** The application pod starts with the `s3-reader` Kubernetes ServiceAccount.

**Step 2:** EKS makes a projected OIDC token available to the pod.

**Step 3:** The AWS SDK uses the token and the IAM role ARN to request credentials from AWS STS.

**Step 4:** STS validates the token using the trusted IAM OIDC provider.

**Step 5:** STS checks the IAM role's trust policy to verify that the token represents the allowed ServiceAccount.

**Step 6:** If validation succeeds, STS returns temporary AWS credentials.

**Step 7:** The application uses those credentials to access S3 according to the IAM role's permissions policy.

This is how IRSA allows Kubernetes workloads to access AWS services without storing static AWS access keys inside pods.

---

# Part 4: Implementing IRSA in AWS EKS

## 5\. Prerequisites

Before starting, make sure you have:

- An existing Amazon EKS cluster.
- AWS CLI installed and configured.
- kubectl installed.
- eksctl installed.
- Permission to create IAM policies, roles and OIDC providers.

For this example, we will use:

| Configuration | Value |
| --- | --- |
| EKS Cluster | my-cluster |
| AWS Region | us-east-1 |
| Namespace | default |
| ServiceAccount | s3-reader |
| IAM Role | s3-read-role |
| S3 Bucket | my-app-data-bucket |

Replace these example values with your actual configuration.

## 6\. Step 1: Find the EKS OIDC issuer URL

First, retrieve the OIDC issuer URL associated with your EKS cluster.

Run:

bash

```bash
aws eks describe-cluster \
  --name my-cluster \
  --region us-east-1 \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

Example output:

text

```text
https://oidc.eks.us-east-1.amazonaws.com/id/EXAMPLE123
```

This is the OIDC issuer URL for your EKS cluster.

Remember:

Every EKS cluster has its own OIDC issuer.

## 7\. Step 2: Associate the OIDC provider with AWS IAM

Now we need to register the EKS OIDC issuer in AWS IAM.

Run:

bash

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --region us-east-1 \
  --approve
```

What happens here?

- eksctl retrieves the EKS cluster's OIDC issuer URL.
- It registers the corresponding OIDC identity provider in AWS IAM.
- AWS IAM can now trust and validate tokens issued by this EKS cluster, subject to the configured trust policies.

You can verify this in:

text

```text
AWS Console
    |
    v
IAM
    |
    v
Identity providers
```

You should see an identity provider similar to:

text

```text
oidc.eks.us-east-1.amazonaws.com/id/EXAMPLE123
```

**Important:** This is generally a one-time setup per EKS cluster.

## 8\. Step 3: Create an IAM policy for S3 access

Now we need to define what our application can do.

Let's create a policy that allows read access to objects in a specific S3 bucket.

Create a file named:

text

```text
s3-read-policy.json
```

Add the following content:

json

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::my-app-data-bucket/*"
    }
  ]
}
```

This policy allows the application to download objects from the specified bucket.

It does not grant permission to upload or delete objects.

Create the IAM policy:

bash

```bash
aws iam create-policy \
  --policy-name EKS-S3-Read-Policy \
  --policy-document file://s3-read-policy.json
```

The output contains the policy ARN.

Example:

text

```text
arn:aws:iam::123456789012:policy/EKS-S3-Read-Policy
```

Save this ARN. We will attach it to the IAM role.

## 9\. Step 4: Create an IAM role with an OIDC trust relationship

This is one of the most important steps.

We need an IAM role that trusts our EKS cluster's OIDC provider.

But we don't want every pod in the cluster to assume this role.

We want only this specific ServiceAccount:

text

```text
namespace: default
serviceaccount: s3-reader
```

First, retrieve the AWS account ID and OIDC issuer:

bash

```bash
export AWS_REGION=us-east-1
export CLUSTER_NAME=my-cluster

export ACCOUNT_ID=$(aws sts get-caller-identity \
  --query Account \
  --output text)

export OIDC_PROVIDER=$(aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --query "cluster.identity.oidc.issuer" \
  --output text | sed 's|^https://||')
```

Now create a file named:

text

```text
trust-policy.json
```

Add:

json

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLE123"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLE123:aud": "sts.amazonaws.com",
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLE123:sub": "system:serviceaccount:default:s3-reader"
        }
      }
    }
  ]
}
```

Replace the example account ID and OIDC provider path with your actual values.

Let's understand this trust policy.

**Principal**

json

```json
"Principal": {
  "Federated": "arn:aws:iam::123456789012:oidc-provider/..."
}
```

This identifies the trusted OIDC identity provider.

**Action**

json

```json
"Action": "sts:AssumeRoleWithWebIdentity"
```

This allows the trusted web identity to request assumption of the IAM role through AWS STS.

**Audience condition**

json

```json
"aud": "sts.amazonaws.com"
```

This ensures the token is intended for AWS STS.

**Subject condition**

json

```json
"sub": "system:serviceaccount:default:s3-reader"
```

This restricts role assumption to the `s3-reader` ServiceAccount in the `default` namespace.

This is an important security feature because we are granting access to a specific Kubernetes workload identity rather than every pod in the cluster.

### Create the IAM role

Run:

bash

```bash
aws iam create-role \
  --role-name s3-read-role \
  --assume-role-policy-document file://trust-policy.json
```

Now attach the S3 permissions policy:

bash

```bash
aws iam attach-role-policy \
  --role-name s3-read-role \
  --policy-arn arn:aws:iam::123456789012:policy/EKS-S3-Read-Policy
```

Now our IAM role has:

- A trust policy that allows the specific EKS ServiceAccount to assume it.
- A permissions policy that allows reading objects from our S3 bucket.

## 10\. Step 5: Create a Kubernetes ServiceAccount

Now we need a Kubernetes ServiceAccount associated with our IAM role.

Create a file named:

text

```text
service-account.yaml
```

Add:

yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/s3-read-role
```

Apply the configuration:

bash

```bash
kubectl apply -f service-account.yaml
```

What does this annotation do?

yaml

```yaml
eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/s3-read-role
```

It tells the EKS IRSA integration which IAM role should be associated with pods using this ServiceAccount.

The EKS admission integration configures the pod with the projected web identity token and the relevant AWS role information.

## 11\. Step 6: Deploy an application using the ServiceAccount

Now let's deploy a pod that uses our ServiceAccount.

Create a file named:

text

```text
deployment.yaml
```

Add:

yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: s3-demo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: s3-demo-app
  template:
    metadata:
      labels:
        app: s3-demo-app
    spec:
      serviceAccountName: s3-reader
      containers:
        - name: aws-cli
          image: amazon/aws-cli:latest
          command: ["sleep", "3600"]
```

Notice this line:

yaml

```yaml
serviceAccountName: s3-reader
```

This is very important.

It tells Kubernetes that the pod must use the `s3-reader` ServiceAccount.

Since the ServiceAccount is associated with the IAM role, the pod can obtain temporary AWS credentials through IRSA.

Deploy the application:

bash

```bash
kubectl apply -f deployment.yaml
```

Check the pod:

bash

```bash
kubectl get pods
```

## 12\. Step 7: Verify AWS credentials inside the pod

Now let's verify whether IRSA is working.

Get the pod name:

bash

```bash
POD_NAME=$(kubectl get pods \
  -l app=s3-demo-app \
  -o jsonpath='{.items[0].metadata.name}')
```

Check the environment variables:

bash

```bash
kubectl exec -it $POD_NAME -- env | grep AWS
```

You should see values similar to:

text

```text
AWS_ROLE_ARN=arn:aws:iam::123456789012:role/s3-read-role

AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

These variables tell the AWS SDK:

- Which IAM role to assume.
- Where to find the projected OIDC token.

The AWS SDK uses these values to obtain temporary credentials from STS.

Now run:

bash

```bash
kubectl exec -it $POD_NAME -- aws sts get-caller-identity
```

Expected output:

json

```json
{
  "UserId": "AROAEXAMPLE:botocore-session-1234567890",
  "Account": "123456789012",
  "Arn": "arn:aws:sts::123456789012:assumed-role/s3-read-role/botocore-session-1234567890"
}
```

Notice the ARN:

text

```text
arn:aws:sts::123456789012:assumed-role/s3-read-role/...
```

This confirms that the application is operating with temporary credentials associated with our `s3-read-role`.

Finally, test S3 access:

bash

```bash
kubectl exec -it $POD_NAME -- \
  aws s3api list-objects-v2 \
  --bucket my-app-data-bucket
```

For this command, the role also needs `s3:ListBucket` permission on the bucket ARN. Our earlier example only granted `s3:GetObject`, so add `s3:ListBucket` to the policy if you want this listing test to succeed. Alternatively, test downloading a known object with `aws s3 cp s3://my-app-data-bucket/path/to/file /tmp/file`.

If the required permissions are configured correctly, your application can access S3 without storing static AWS credentials inside the pod.

---

# Part 5: Final Architecture and Important Concepts

## 13\. Complete IRSA architecture

Let's summarize the entire implementation.

text

```text
                    AWS ACCOUNT
                         |
                         |
                 EKS OIDC ISSUER
                         |
                         |
                  IAM OIDC PROVIDER
                         |
                         |
       -----------------------------------
       |                                 |
       |                                 |
   EKS CLUSTER                      AWS IAM
       |                                 |
       |                            IAM ROLE
       |                           s3-read-role
       |                                 |
       |                         Trust Policy
       |                         Permissions Policy
       |                                 |
       |                                 |
   APPLICATION POD                       |
       |                                 |
   ServiceAccount                        |
     s3-reader                            |
       |                                 |
   Projected OIDC Token                   |
       |                                 |
       |                                 |
       ----------- AWS STS ---------------
                    |
                    |
       AssumeRoleWithWebIdentity
                    |
                    |
          Validate OIDC Token
                    |
                    |
           Evaluate Trust Policy
                    |
                    |
          Return Temporary Credentials
                    |
                    |
                    v
                AMAZON S3
```

## 14\. Difference between the IAM trust policy and permissions policy

This is a very common interview question.

| Trust Policy | Permissions Policy |
| --- | --- |
| Defines who can assume the role | Defines what the role can do |
| Uses OIDC provider as federated principal | Uses AWS service actions and resources |
| Contains `sts:AssumeRoleWithWebIdentity` | Contains actions such as `s3:GetObject` |
| Validates workload identity through conditions | Authorizes AWS resource access |

For example:

- Trust policy: Only the `default/s3-reader` ServiceAccount can assume this role.
- Permissions policy: The assumed role can read objects from a particular S3 bucket.

Both are necessary for IRSA.

## 15\. Important questions

**Q1. What is OIDC?**

OIDC is an authentication protocol built on OAuth 2.0 that allows one system to verify the identity of another using identity tokens.

**Q2. Why do we need an OIDC provider in EKS?**

AWS IAM needs a trusted identity provider to validate the OIDC tokens issued by EKS. This enables AWS STS to authenticate Kubernetes workloads.

**Q3. What is the role of AWS STS in IRSA?**

STS validates the workload's web identity token, evaluates the IAM role's trust policy and returns temporary AWS credentials when the request is authorized.

**Q4. What is the difference between an OIDC provider and an IAM role?**

The OIDC provider establishes which token issuer AWS trusts. The IAM role defines which identities can assume it and what permissions they receive.

**Q5. Why do we use ServiceAccounts?**

Kubernetes ServiceAccounts provide identities for workloads. IRSA uses the ServiceAccount identity to associate a specific Kubernetes workload with an AWS IAM role.

**Q6. Does IRSA use permanent AWS credentials?**

No. IRSA uses OIDC tokens and AWS STS to obtain temporary AWS credentials.

**Q7. Can every pod in an EKS cluster access the IAM role?**

No. The trust policy can restrict access to a specific namespace and ServiceAccount. Only pods using an authorized ServiceAccount can assume that role, subject to the other security controls.

---

# Conclusion

OIDC provides a way to establish trust between two systems using tokens.

In AWS EKS, this concept is used by IRSA to authenticate Kubernetes workloads and provide them with temporary AWS credentials.

The complete process is:

1. EKS issues an OIDC token representing a Kubernetes ServiceAccount.
2. AWS IAM trusts the EKS OIDC issuer through an IAM OIDC provider.
3. The pod presents its token to AWS STS.
4. STS validates the token and checks the IAM role's trust policy.
5. STS returns temporary credentials for the IAM role.
6. The application uses those credentials to access AWS services.

**Remember this simple relationship:**

- EKS identifies the workload.
- OIDC provides the workload identity token.
- IAM OIDC provider establishes trust in the token issuer.
- IAM role defines trust and permissions.
- STS exchanges the token for temporary credentials.
- AWS services authorize requests made with those credentials.

This is the foundation of IAM Roles for Service Accounts (IRSA) in Amazon EKS.

## Recommended references

These are official AWS documentation pages that support the implementation:

1. [AWS – IAM Roles for Service Accounts](<https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html?utm_source=chatgpt.com>)
2. [AWS – Create an IAM OIDC Provider for Your Cluster](<https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html?sc_campaign=appswave&sc_channel=el&sc_content=eks-cluster-load-balancer-ipv4&sc_country=mult&sc_geo=mult&sc_outcome=acq&utm_source=chatgpt.com>)
3. [AWS – Assign IAM Roles to Kubernetes Service Accounts](<https://docs.aws.amazon.com/eks/latest/userguide/associate-service-account-role.html?utm_source=chatgpt.com>)
4. [eksctl – IAM Roles for Service Accounts](<https://docs.aws.amazon.com/eks/latest/eksctl/iamserviceaccounts.html?utm_source=chatgpt.com>)

A couple of important publishing notes:

- The article uses manual IAM role creation so readers understand the trust policy and permissions policy separately.
- The `eksctl create iamserviceaccount` command can automate much of that configuration, but I recommend keeping the manual explanation because it teaches the underlying architecture.
- In production, use narrowly scoped IAM permissions, recent AWS SDK versions, and restrict access to EC2 instance metadata where appropriate.s