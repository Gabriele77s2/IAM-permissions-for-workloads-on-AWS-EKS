# Lab 2 - Configuring EKS Pod Identity

You want your applications in the green cluster to access S3. But you don't want to use the EC2 instance roles. There are two ways to accomplish this:

-   IAM roles for service accounts
-   EKS Pod Identities

In this lab 2, we will focus on implementing it using, the newer and recommended version, EKS Pod Identities.

[

## EKS Pod Identity at a glance

](https://catalog.us-east-1.prod.workshops.aws/workshops/7c41ed45-0120-4af8-bd23-420f9f0f5eb1/en-US/getting-started#eks-pod-identity-at-a-glance)

The overall workflow to start using Amazon EKS Pod Identity can be summarized in a few simple steps:

-   **Step 1**: Install Amazon EKS Pod Identity Agent add-on using the Amazon EKS console or AWS Command Line Interface (AWS CLI).
-   **Step 2**: Create an IAM role with required permissions for your application and specify `pods.eks.amazonaws.com` as the service principal in its trust policy.
-   **Step 3**: Map the role to a service account directly in the Amazon EKS console, APIs, or AWS CLI.

[

### Step 1

](https://catalog.us-east-1.prod.workshops.aws/workshops/7c41ed45-0120-4af8-bd23-420f9f0f5eb1/en-US/getting-started#step-1)

In lab 1, you saw both of your clusters already have Amazon EKS Pod Identity Agent Add-on installed.

![addon](AWS%20Identity%20and%20Access%20Management%20permissions%20to%20workloads%20on%20Amazon%20Elastic%20Kubernetes%20Service/add-on.png)

If they don't, you can easily install it using EKS console, or AWS CLI:

```shell
1 2 3 4 aws eks create-addon \ --cluster-name cluster-name \ --addon-name eks-pod-identity-agent \ --addon-version v1.0.0-eksbuild.1
```

After installing the add-on installed, you can verify it using kubectl. Your cluster should have 2 eks-pod-identity-agent pods running.

```shell
1 kubectl get daemonset -n kube-system
```

That is it, **Step 1** is now complete.

[

### Step 2

](https://catalog.us-east-1.prod.workshops.aws/workshops/7c41ed45-0120-4af8-bd23-420f9f0f5eb1/en-US/getting-started#step-2)

Instead of creating this role from scratch, Navigate to [IAM console](https://console.aws.amazon.com/iam/home)  and find the **eks-pod-identity-green-cluster** role.

You will find it pre-created with a permission policy that allows access to S3. ![podrole](AWS%20Identity%20and%20Access%20Management%20permissions%20to%20workloads%20on%20Amazon%20Elastic%20Kubernetes%20Service/podrole.png)

In its trusted entities, you will find `pods.eks.amazonaws.com`. ![trust](AWS%20Identity%20and%20Access%20Management%20permissions%20to%20workloads%20on%20Amazon%20Elastic%20Kubernetes%20Service/podtrust.png)

Now all it is missing is a mapping to a k8s service account.

[

### Step 3

](https://catalog.us-east-1.prod.workshops.aws/workshops/7c41ed45-0120-4af8-bd23-420f9f0f5eb1/en-US/getting-started#step-3)

Navigate to [EKS console](https://console.aws.amazon.com/eks/home) . Select your green cluster, and navigate to the Access section. Under Pod Identity associations, you will find an association between the **eks-pod-identity-green-cluster IAM role**, the **default k8s namespace**, and a **sa-eks-pod-identity-demo service account**.

![greenidentity](AWS%20Identity%20and%20Access%20Management%20permissions%20to%20workloads%20on%20Amazon%20Elastic%20Kubernetes%20Service/pod-identity-green.png)

Since the association is already created, you only need to associate the `sa-eks-pod-identity-demo` service account to a pod.

Back at your Cloud9 console, verify if this service account already exists using kubectl:

```shell
1 kubectl get serviceAccounts
```

It doesn't.

```
<span>$ kubectl get serviceAccounts
</span>NAME                       SECRETS   AGE
default                    0         4h20m
```

Create a manifest with the service account, and a new sample application pod.

```shell
1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 cat <<-EOT > demo-eks-pod-identity.json apiVersion: v1 kind: ServiceAccount metadata: name: sa-eks-pod-identity-demo --- apiVersion: v1 kind: Pod metadata: name: reinf24-eks-pod-identity-demo spec: serviceAccountName: sa-eks-pod-identity-demo containers: - name: reinf24-eks-pod-identity-demo image: amazon/aws-cli:latest command: ["/bin/bash", "-c", "--"] args: ["while true; do sleep 5; done;"] EOT
```

Deploy the manifest using kubectl

```shell
1 kubectl apply -f ./demo-eks-pod-identity.json
```

After a few seconds, connect to your newly created pod and test if it can access S3.

```
<span>$ kubectl exec -it reinf24-eks-pod-identity-demo -- /bin/bash
</span>bash-4.2# aws sts get-caller-identity
{
    "UserId": "AROASOCZAIW3FEGWIW5IP:eks-eks-pod-id-reinf24-ek-9db56306-a7ee-4cae-b7a7-fdf6978571dd",
    "Account": "111122223333",
    "Arn": "arn:aws:sts::111122223333:assumed-role/eks-pod-identity-green-cluster/eks-eks-pod-id-reinf24-ek-9db56306-a7ee-4cae-b7a7-fdf6978571dd"
}
bash-4.2# aws s3 ls
```

It can access S3, and it is using the **eks-pod-identity-green-cluster** IAM role. Recall you did not change the hop-limit for IMDS. EKS Pod Identities works with a hop-limit of 1.

[

## Under the hood of EKS Pod Identity

](https://catalog.us-east-1.prod.workshops.aws/workshops/7c41ed45-0120-4af8-bd23-420f9f0f5eb1/en-US/getting-started#under-the-hood-of-eks-pod-identity)

You managed to give the pod in the green cluster access to S3. But how does it work under the hood?

1.  Amazon EKS cluster / IAM administrator creates an IAM role that can be assumed by the newly introduced EKS Service principal `pods.eks.amazonaws.com`.
2.  After the AWS IAM role creation, Amazon EKS cluster administrator creates an association between the IAM role and Kubernetes service account.
3.  After the AWS IAM role is associated with the service account, any newly created pods using that service account will be intercepted by the EKS Pod Identity webhook. This webhook runs on the Amazon EKS cluster’s control plane, and is fully managed by EKS. The webhook mutates the pod spec as shown below with the bolded environment variables. All AWS Software Development Kits (SDKs) have a series of places (or sources) that it checks in order to find valid credentials to use, to make a request to an AWS service. After credentials are found, the search is stopped. This systematic search is called the credential provider chain. For EKS Pod Identity, we leverage the HTTP credential provider mechanism that is already built into AWS SDKs and CLI to retrieve credentials from an HTTP endpoint specified in the environment variable `AWS_CONTAINER_CREDENTIALS_FULL_URI`. This endpoint is served by the EKS Pod Identity Agent running on the worker node, we will talk more about this in the next step. The location to the projected JWT token that is used to exchange for IAM credentials is specified in the environment variable `AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE`.
4.  AWS SDK/CLI calls the Amazon EKS Pod Identity Agent endpoint to retrieve the temporary IAM credentials. EKS Pod Identity Agent runs as a DaemonSet pod on every eligible worker node. This agent is made available to you as an EKS Add-on and is a pre-requisite to use EKS Pod Identity feature. EKS Auth API (AssumeRoleForPodIdentity) decodes the JWT token and validates the role associations with the service account. After successful validation, EKS Auth API will return the temporary AWS credentials. It will also set session tags such as kubernetes-namespace, kubernetes-service-account, eks-cluster-arn, eks-cluster-name, kubernetes-pod-name, kubernetes-pod-uid.
5.  AWS SDKs will use the vended temporary credentials to access other AWS resources.

![flow](AWS%20Identity%20and%20Access%20Management%20permissions%20to%20workloads%20on%20Amazon%20Elastic%20Kubernetes%20Service/flow.png)

[Amazon EKS Pod Identity: a new way for applications on EKS to obtain IAM credentials](https://aws.amazon.com/blogs/containers/amazon-eks-pod-identity-a-new-way-for-applications-on-eks-to-obtain-iam-credentials/)  blog post explains the process in detail.

Let's drill down into step 3.

Any of the supported AWS SDKs will automatically detect that you have enabled Pod Identity and start using it. How does this process work?

All SDKs have a series of places (or sources) that they check in order to find valid credentials to use to make a request to an AWS service. After valid credentials are found, the search is stopped. This systematic search is called the default credential provider chain.

The following figure shows examples of credentials sources.

![creds](AWS%20Identity%20and%20Access%20Management%20permissions%20to%20workloads%20on%20Amazon%20Elastic%20Kubernetes%20Service/cred.png)

In this case, the existing in-cluster mutating admission controller amazon-eks-pod-identity-webhook was updated to automatically inject the `AWS_CONTAINER_CREDENTIALS_FULL_URI` and `AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE` environment variables into pods.

In your green cluster, verify if these were injected in your pods.

```shell
1 kubectl get pod/reinf24-eks-pod-identity-demo -o yaml | grep -A 10 env
```

```shell
1 kubectl get pod/reinf24-imds-only -o yaml | grep -A 10 env
```

You can tell the difference. Only the pod configured to use EKS Identity has these variables configured.

[

## On your own

](https://catalog.us-east-1.prod.workshops.aws/workshops/7c41ed45-0120-4af8-bd23-420f9f0f5eb1/en-US/getting-started#on-your-own)

Deploy EKS Pod identity to the red EKS cluster, and verify the pod is using the pod instead of the EC2 role. Use the **eks-pod-identity-green-cluster** IAM role.

[

### Tips

](https://catalog.us-east-1.prod.workshops.aws/workshops/7c41ed45-0120-4af8-bd23-420f9f0f5eb1/en-US/getting-started#tips)

-   Use **aws sts get-caller-identity** to verify what role is being used
-   Deploy a new pod instead of using the old one

If you are stuck, ask support from your instructor.
