# Lab 3 - Restrict access using Attribute-based access control (ABAC)

EKS Pod Identity attaches tags to the temporary credentials to each pod with attributes such as cluster name, namespace, service account name. These role session tags enable administrators to author a single role that can work across service accounts by allowing access to AWS resources based on matching tags. By adding support for role session tags, customers can enforce tighter security boundaries between clusters, and workloads within clusters, while reusing the same IAM roles and IAM policies.

The following list contains all of the keys for tags that are added to the AssumeRole request made by Amazon EKS. To use these tags in policies, use **${aws:PrincipalTag/** followed by the key, for example **${aws:PrincipalTag/kubernetes-namespace}**.

-   eks-cluster-arn
-   eks-cluster-name
-   kubernetes-namespace
-   kubernetes-service-account
-   kubernetes-pod-name
-   kubernetes-pod-uid

[

## Implement service-account-based access to S3 objects

](https://catalog.us-east-1.prod.workshops.aws/workshops/7c41ed45-0120-4af8-bd23-420f9f0f5eb1/en-US/lab-3#implement-service-account-based-access-to-s3-objects)

Navigate to [S3 console](https://console.aws.amazon.com/s3/home)  and create a S3 bucket.

Populate it with an object, any object. Tag your newly created S3 object with the tag **kubernetes-service-account = abac**.

Navigate back to your Cloud9 instance. In your green cluster, create:

-   two service accounts linked to eks-pod-identity-abac IAM role
-   two pods each using a different service account

Do you need some help with the resolution?

Use the kubectl to create the service accounts. A manifest also works.

```shell
1 2 kubectl create serviceaccount abac kubectl create serviceaccount abac-dev
```

You can create the associations between the service accounts and the role using EKS Console as shown below: ![identity](AWS%20Identity%20and%20Access%20Management%20permissions%20to%20workloads%20on%20Amazon%20Elastic%20Kubernetes%20Service/createpodidentity.png) Replace the values with your own.

Lastly, create the two pods. kubectl works, but using a manifest is easier.

```shell
1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 cat <<-EOT > abac-eks-pod-identity.json --- apiVersion: v1 kind: Pod metadata: name: reinf24-eks-pod-identity-abac spec: serviceAccountName: abac containers: - name: reinf24-eks-pod-identity-abac image: amazon/aws-cli:latest command: ["/bin/bash", "-c", "--"] args: ["while true; do sleep 5; done;"] --- apiVersion: v1 kind: Pod metadata: name: reinf24-eks-pod-identity-abac-dev spec: serviceAccountName: abac-dev containers: - name: reinf24-eks-pod-identity-abac-dev image: amazon/aws-cli:latest command: ["/bin/bash", "-c", "--"] args: ["while true; do sleep 5; done;"] EOT kubectl apply -f ./abac-eks-pod-identity.json
```

Verify if you can access the S3 object from each of your newly created pods. It's expected that only the pod with the abac service account can access it.

Do you need some help with the verifications?

You can verify what role your pods are using with the following command:

```shell
1 aws sts get-caller-identity
```

```
<span>{
</span>    "UserId": "AROAU2FFHLCOYBKN6Z2RY:eks-eks-pod-id-reinf24-ek-16f667fa-2208-4b73-b327-cf12442e874b",
    "Account": "111122223333",
    "Arn": "arn:aws:sts::111122223333:assumed-role/eks-pod-identity-abac/eks-eks-pod-id-reinf24-ek-16f667fa-2208-4b73-b327-cf12442e874b"
}
```

You must execute it from inside the pods. You can use kubectl to access a bash.

```shell
1 kubectl exec -it reinf24-eks-pod-identity-abac -- /bin/bash
```

To copy your file from S3, you can use the following syntax

```shell
1 aws s3 cp s3://<BUCKET NAME>/<OBJECT NAME> file
```

If you want to learn more about this feature visit [Define permissions for EKS Pod Identities to assume roles based on tags](https://docs.aws.amazon.com/eks/latest/userguide/pod-id-abac.html) .
