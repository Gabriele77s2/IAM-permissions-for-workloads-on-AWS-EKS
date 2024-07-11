# Lab 1 - Troubleshooting application roles

Navigate to [Cloud9 console](https://console.aws.amazon.com/cloud9/home) 

[

## Cloud9 IAM Setup

](https://catalog.us-east-1.prod.workshops.aws/workshops/7c41ed45-0120-4af8-bd23-420f9f0f5eb1/en-US/lab-1#cloud9-iam-setup)

In order to perform some operations for this workshop, it is necessary to use the IAM role associated with your Cloud9 environment. In order for this role to take effect, it is necessary to disable Cloud9's normal management of your credentials (AWS managed temporary credentials). To do this, run the following command on Cloud9's terminal.

```shell
1 chmod 000 ~/.aws/credentials
```

Now, you are ready to move on to the fun part.

[

## Explore the EKS setup

](https://catalog.us-east-1.prod.workshops.aws/workshops/7c41ed45-0120-4af8-bd23-420f9f0f5eb1/en-US/lab-1#explore-the-eks-setup)

Navigate to [EKS console](https://console.aws.amazon.com/eks/home)  There you will find two clusters as shown in the following figure. ![k8s](AWS%20Identity%20and%20Access%20Management%20permissions%20to%20workloads%20on%20Amazon%20Elastic%20Kubernetes%20Service/eks-clusters.png)

Explore the configurations of each of these clusters.

Navigate to [EC2 console](https://console.aws.amazon.com/ec2/home)  There you will find 5 EC2 instances as shown in the following figure. ![ec2s](AWS%20Identity%20and%20Access%20Management%20permissions%20to%20workloads%20on%20Amazon%20Elastic%20Kubernetes%20Service/ec2s.png)

Explore the roles associated with each EC2 instance that belong to the clusters. The following figure shows the role assigment from an EC2 of the red cluster. ![ec2role](AWS%20Identity%20and%20Access%20Management%20permissions%20to%20workloads%20on%20Amazon%20Elastic%20Kubernetes%20Service/ec2role.png)

Verify the policy assigment of those roles, they should have S3 access. ![policyec2](AWS%20Identity%20and%20Access%20Management%20permissions%20to%20workloads%20on%20Amazon%20Elastic%20Kubernetes%20Service/policyec2.png)

Return to [EKS console](https://console.aws.amazon.com/eks/home) , and verify that Amazon EKS Pod Identity Agent add-on is already pre-installed in both EKS clusters. ![addon](AWS%20Identity%20and%20Access%20Management%20permissions%20to%20workloads%20on%20Amazon%20Elastic%20Kubernetes%20Service/add-on.png)

Now that you understand the resources available, you will deploy pods to each of these EKS clusters and verify their access.

[

## Deploy test applications

](https://catalog.us-east-1.prod.workshops.aws/workshops/7c41ed45-0120-4af8-bd23-420f9f0f5eb1/en-US/lab-1#deploy-test-applications)

Return to [Cloud9 console](https://console.aws.amazon.com/cloud9/home) . Use Cloud9 bash terminal for the following sections.

[

### Install kubectl

](https://catalog.us-east-1.prod.workshops.aws/workshops/7c41ed45-0120-4af8-bd23-420f9f0f5eb1/en-US/lab-1#install-kubectl)

```shell
1 2 3 curl -LO https://dl.k8s.io/release/v1.29.0/bin/linux/amd64/kubectl chmod +x ./kubectl sudo mv ./kubectl /usr/local/bin/kubectl
```

Now check the version of kubectl.

The output should be similar to:

```
<span>$ kubectl version --client
</span>Client Version: v1.29.0
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
```

[

### Connect to your red EKS cluster

](https://catalog.us-east-1.prod.workshops.aws/workshops/7c41ed45-0120-4af8-bd23-420f9f0f5eb1/en-US/lab-1#connect-to-your-red-eks-cluster)

Define the region you are doing the workshop in, and add your EKS context:

```
<span>aws eks update-kubeconfig --name eks-pod-identity-red</span>
```

If you are successful, it will output:

```
<span>Added new context arn:aws:eks:us-east-1:111122223333:cluster/eks-pod-identity-red to /home/ssm-user/.kube/config</span>
```

Now verify what pods are already running using kubectl:

```
<span>kubectl get pods --all-namespaces -o wide</span>
```

You should see 9 pods. Notice 2 of them are part of EKS Pod Identity Agent add-on.

```
<span>NAMESPACE     NAME                           READY   STATUS    RESTARTS   AGE     IP             NODE                           NOMINATED NODE   READINESS GATES
</span>kube-system   aws-node-hjbq8                 2/2     Running   0          8m12s   10.0.255.81    ip-10-0-255-81.ec2.internal    &lt;none&gt;           &lt;none&gt;
kube-system   aws-node-snntc                 2/2     Running   0          8m8s    10.0.254.238   ip-10-0-254-238.ec2.internal   &lt;none&gt;           &lt;none&gt;
kube-system   coredns-54d6f577c6-cg2k2       1/1     Running   0          13m     10.0.255.204   ip-10-0-255-81.ec2.internal    &lt;none&gt;           &lt;none&gt;
kube-system   coredns-54d6f577c6-czc72       1/1     Running   0          13m     10.0.255.4     ip-10-0-255-81.ec2.internal    &lt;none&gt;           &lt;none&gt;
kube-system   eks-pod-identity-agent-2lzhb   1/1     Running   0          8m12s   10.0.255.81    ip-10-0-255-81.ec2.internal    &lt;none&gt;           &lt;none&gt;
kube-system   eks-pod-identity-agent-ztkst   1/1     Running   0          8m8s    10.0.254.238   ip-10-0-254-238.ec2.internal   &lt;none&gt;           &lt;none&gt;
kube-system   kube-proxy-qgqd9               1/1     Running   0          8m8s    10.0.254.238   ip-10-0-254-238.ec2.internal   &lt;none&gt;           &lt;none&gt;
kube-system   kube-proxy-tx877               1/1     Running   0          8m12s   10.0.255.81    ip-10-0-255-81.ec2.internal    &lt;none&gt;           &lt;none&gt;
```

[

### Deploy a sample pod in the red cluster

](https://catalog.us-east-1.prod.workshops.aws/workshops/7c41ed45-0120-4af8-bd23-420f9f0f5eb1/en-US/lab-1#deploy-a-sample-pod-in-the-red-cluster)

Create a kubernetes manifest for a sample pod. This pod uses amazon/aws-cli container image.

```shell
1 2 3 4 5 6 7 8 9 10 11 12 cat <<-EOT > demo-imds.json apiVersion: v1 kind: Pod metadata: name: reinf24-imds-only spec: containers: - name: reinf24-imds-only image: amazon/aws-cli:latest command: ["/bin/bash", "-c", "--"] args: ["while true; do sleep 5; done;"] EOT
```

Deploy the pod using kubectl.

```
<span>kubectl apply -f ./demo-imds.json</span>
```

After a few seconds, connect to your newly created pod, and verify if it can list S3 buckets.

```shell
1 2 3 kubectl exec -it reinf24-imds-only -- /bin/bash aws sts get-caller-identity aws s3 ls
```

It should execute successfully as shown below. Although there are no S3 buckets.

```shell
1 2 3 4 5 6 7 8 9 $ kubectl exec -it reinf24-imds-only -- /bin/bash bash-4.2# aws sts get-caller-identity { "UserId": "AROASOCZAIW3D67TEMWOW:i-00e75a61dd693b28c", "Account": "111122223333", "Arn": "arn:aws:sts::111122223333:assumed-role/eks-nodes-RedCluster/i-00e75a61dd693b28c" } bash-4.2# aws s3 ls
```

The pod is using eks-nodes-RedCluster role. Exit the pod.

Why does this work? Before finding out why, try the same, this time in the green cluster.

[

### Deploy a sample pod in the green cluster

](https://catalog.us-east-1.prod.workshops.aws/workshops/7c41ed45-0120-4af8-bd23-420f9f0f5eb1/en-US/lab-1#deploy-a-sample-pod-in-the-green-cluster)

Change to your green EKS context:

```
<span>aws eks update-kubeconfig --name eks-pod-identity-green</span>
```

If you are successful, it will output:

```
<span>Updated context arn:aws:eks:us-east-1:111122223333:cluster/eks-pod-identity-green in /home/ec2-user/.kube/config</span>
```

Verify what pods are already running using kubectl. It will output something similar to the red cluster in the previous section.

Deploy the same pod using kubectl on this cluster.

```
<span>kubectl apply -f ./demo-imds.json</span>
```

After a few seconds, connect to your newly created pod, and verify if it can list S3 buckets.

```shell
1 2 3 kubectl exec -it reinf24-imds-only -- /bin/bash aws sts get-caller-identity aws s3 ls
```

The output should look like this:

```
<span>bash-4.2# aws sts get-caller-identity
</span>Unable to locate credentials. You can configure credentials by running "aws configure".
bash-4.2# aws s3 ls
Unable to locate credentials. You can configure credentials by running "aws configure".
```

It fails. The pod cannot locate credentials to make the request. Two similar clusters, one succeeds, another fails. Why?

[

### How Instance Metadata Service Version 2 works

](https://catalog.us-east-1.prod.workshops.aws/workshops/7c41ed45-0120-4af8-bd23-420f9f0f5eb1/en-US/lab-1#how-instance-metadata-service-version-2-works)

Although these clusters are similar, they have one difference; **IMDS hop limit**.

The red cluster instances are configured with a IMDS hop limit of 2, while the green cluster instances have a IMDS hop limit of 1.

This is not evident using the AWS console, but you can verify it using the CLI.

Navigate to [EC2 console](https://console.aws.amazon.com/ec/home)  and note down the ids of one of your red instances, and one of your green instances.

Return to [Cloud9 console](https://console.aws.amazon.com/cloud9/home) . Run the following command, twice. Once for each instance ID:

```shell
1 2 3 aws ec2 describe-instances \ --query 'Reservations[].Instances[].[InstanceId, MetadataOptions, Tags[?Key==Name]]' \ --instance-ids <INSTANCE IDs>
```

Under `MetadataOptions`, compare the values of `HttpPutResponseHopLimit`. You can read more on [EC2 Instance Metadata configuration](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-options.html)  and on how [IMDSv2 works](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-metadata-v2-how-it-works.html) .

What is happening in this scenario is that the sample pod inside the red cluster is able to take advantage of your IAM role that's attached to your EC2 nodes, which in turn has access to S3 as shown in the following figure. While the sample pod in the green cluster cannot.

![policyec2](AWS%20Identity%20and%20Access%20Management%20permissions%20to%20workloads%20on%20Amazon%20Elastic%20Kubernetes%20Service/policyec2.png)

[

#### How does the hop limit influence the pod ability to use the instance role?

](https://catalog.us-east-1.prod.workshops.aws/workshops/7c41ed45-0120-4af8-bd23-420f9f0f5eb1/en-US/lab-1#how-does-the-hop-limit-influence-the-pod-ability-to-use-the-instance-role)

Each instance that you launch has an instance identity role that represents its identity. An instance identity role is a type of IAM role. The instance identity role credentials are accessible from the Instance Metadata Service (IMDS). In a container environment, if the hop limit is 1, the IMDSv2 response does not return because going to the container is considered an additional network hop.

[

## Summary

](https://catalog.us-east-1.prod.workshops.aws/workshops/7c41ed45-0120-4af8-bd23-420f9f0f5eb1/en-US/lab-1#summary)

It is a best practice to [Use one IAM role per application](https://aws.github.io/aws-eks-best-practices/security/docs/iam/#use-one-iam-role-per-application)  instead of more generic roles on the managed EC2 instances. Image the scenario you have multiple applications in the same cluster, and each of those needs access to different AWS services. For example one needs access to DynamoDB, while the other needs access to RDS. If you have a single role on the EC2 that allows access to both these services, each application would have more access than it requires. How can you achieve this?

Find out in Lab 2 - Configuring EKS Pod Identity.
