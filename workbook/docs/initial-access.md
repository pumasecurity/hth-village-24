# Kubernetes Initial Access

Microsoft's Threat Matrix for Kubernetes highlights a number of [Initial Access](https://microsoft.github.io/Threat-Matrix-for-Kubernetes/tactics/InitialAccess/){: target="_blank" rel="noopener"} techniques used by attackers to compromise a cluster. One of the techniques, documented as [Using cloud credentials](https://microsoft.github.io/Threat-Matrix-for-Kubernetes/techniques/Using%20Cloud%20Credentials/){: target="_blank" rel="noopener"}, occurs when cloud credentials are stolen or unintentionally leaked.

## AWS CLI Configuration

The AWS Command Line Interface (CLI) provides programmatic access to the AWS service APIs. To gain access to the AWS Elastic Kubernetes Cluster (EKS), you need to configure the AWS CLI to use the stolen credentials that you received from the village hosts. Set the required [AWS CLI environment variables](https://docs.aws.amazon.com/cli/v1/userguide/cli-configure-envvars.html){: target="_blank" rel="noopener"} to use the stolen credentials. What is the name of the compromised AWS principal?

??? tip "Hint"

    - The AWS CLI checks several locations for credentials when authenticating to the AWS APIs. This includes a local configuration file (~/.aws/credentials), environment variables, and the instance metadata service when running inside an AWS process (EC2, Lambda, etc.). Set the `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_DEFAULT_REGION` environment variables to the stolen credentials.

        ```bash
        export AWS_ACCESS_KEY_ID=STOLEN_ACCESS_KEY
        export AWS_SECRET_ACCESS_KEY=STOLEN_SECRET_KEY
        export AWS_DEFAULT_REGION=STOLEN_REGION
        ```

    - The AWS Security Token Service (STS) has a [GetCallerIdentity](https://docs.aws.amazon.com/STS/latest/APIReference/API_GetCallerIdentity.html){: target="_blank" rel="noopener"} that returns details about the IAM user or role calling the API. Use the `aws sts get-caller-identity` command to retrieve the compromised principal's name and AWS account id.

        ```bash
        aws sts get-caller-identity
        ```

        !!! abstract "Expected Output"
            ```json hl_lines="4"
            {
              "UserId": "AIDA2ZCFBDI7W52ZHQYQ7",
              "Account": "123456789012",
              "Arn": "arn:aws:iam::123456789012:user/?????"
            }
            ```

??? success "Answer"

    - The compromised principal's name is in the **Arn** field in the output, which identifies an IAM user named **kubeace-maverick** followed by a random identifier.

        ```
        kubeace-maverick-randomid
        ```

## AWS EKS Initial Access

Amazon's Elastic Kubernetes Service (EKS) is a managed Kubernetes service that makes it simple to run Kubernetes on AWS without needing to install, configure, and maintain your own Kubernetes control plane. EKS is integrated with many AWS services, including the [Identity and Access Management (IAM)](https://docs.aws.amazon.com/eks/latest/userguide/cluster-auth.html){: target="_blank" rel="noopener"} service.

With the appropriate [EKS access entry](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html){: target="_blank" rel="noopener"} and permissions, AWS principals can configure `kubectl` to authenticate directly to the EKS cluster.

1. Use the AWS CLI to search the AWS account for EKS clusters. What is the name of the EKS cluster that you need to access?

    ??? tip "Hint"

        - The `aws eks list-clusters` command lists the EKS clusters in the specified AWS account.

            ```bash
            aws eks list-clusters
            ```

            !!! abstract "Expected Output"
                ```json hl_lines="3"
                {
                  "clusters": [
                    "?????"
                  ]
                }
                ```

    ??? success "Answer"

        - The EKS cluster you need to access is named **hth-eks-cluster**.

            ```
            hth-eks-cluster
            ```

1. Use AWS CLI to update your machine's [kubeconfig file](https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html){: target="_blank" rel="noopener"} and access the EKS cluster. What Kubernetes group(s) is the compromised AWS principal a member of?

    ??? tip "Hint"

        - The `aws eks update-kubeconfig` command updates the `~/.kube/config` file with details `kubectl` needs to authenticate to the `hth-eks-cluster` cluster.

            ```bash
            aws eks update-kubeconfig --name hth-eks-cluster
            ```

            !!! abstract "Expected Output"
                ```json hl_lines="4"
                Updated context arn:aws:eks:us-west-2:123456789012:cluster/hth-eks-cluster in /Users/user/.kube/config
                ```

        - Similar to the `aws sts get-caller-identity` command, `kubectl` has its own command that returns details about the authenticated user. Use the `kubectl auth whoami` command to view the compromised principal's group memberships.

            ```bash
            kubectl auth whoami
            ```

            !!! abstract "Expected Output"
                ```
                ATTRIBUTE                                              VALUE
                Username                                               kubeace-maverick-randomid
                UID                                                    aws-iam-authenticator:123456789012:AIDA2ZCFBDI7W52ZHQYQ7
                Groups                                                 [?????]
                Extra: accessKeyId                                     [AKIA2ZCFBDI74CS5EG5L]
                Extra: arn                                             [arn:aws:iam::123456789012:user/kubeace-maverick-randomid]
                Extra: canonicalArn                                    [arn:aws:iam::123456789012:user/kubeace-maverick-randomid]
                Extra: principalId                                     [AIDA2ZCFBDI7W52ZHQYQ7]
                Extra: sessionName                                     []
                Extra: sigs.k8s.io/aws-iam-authenticator/principalId   [AIDA2ZCFBDI7W52ZHQYQ7]
                ```

    ??? success "Answer"

        - The compromised principal is a member of the **hth-data-viewers** and **system:authenticated** groups. Based on the name of the first group (`hth-data-viewers`), it is likely that the principal has read-only access to resources in the cluster that live in the `hth` namespace.

            ```
            Groups [hth-data-viewers system:authenticated]
            ```

## Village Challenges

Now that you have successfully gained access to the EKS cluster using the compromised AWS credentials, it is time to start elevating privileges inside the cluster and exfiltrating HTH data. Navigate to the [Airborneio-24 Challenge](./airborneio-24.md) to find the first flag.
