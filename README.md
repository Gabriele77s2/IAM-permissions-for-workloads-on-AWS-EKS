# IAM-permissions-for-workloads-on-AWS-EKS

The activity is organized into three labs:

- Lab 1 - Troubleshooting application roles
- Lab 2 - Configuring EKS Pod Identity
- Lab 3 - Restrict access using Attribute-based access control (ABAC)

During the first lab you are going to troubleshoot EKS pods access to AWS services, in this case S3. While troubleshooting, you will learn how you can define IAM access permissions at different levels (e.g. EC2 node).

In the second lab, you will fix the pods without S3 access in the first lab by configuring EKS Pod Identity, and granting them IAM permissions.

In the third and last lab, you scope your S3 access using ABAC.
