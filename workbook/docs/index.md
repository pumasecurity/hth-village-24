# Kubernetes Security Village

Welcome to the Hackers Teaching Hackers (HTH) 2024 Kubernetes Security Village.

![](https://images.squarespace-cdn.com/content/v1/5e1d245285d6d83067bd591f/bc18dbe4-d21e-4123-8728-ace94000739b/DALL%C2%B7E+2024-07-06+15.51.19+-+A+banner+promotional+image+for+a+cyber+security+conference+called+%27HTH+2024%27+with+a+Terminator-inspired+theme.+The+design+features+a+dark%2C+high-tech+b.jpg?format=2500w)

The Kubernetes Security Village explores a few of the attacker techniques covered by the [Microsoft Threat Matrix for Kubernetes](https://microsoft.github.io/Threat-Matrix-for-Kubernetes/){: target="_blank" rel="noopener"}.

* Initial Access - Techniques used to gain a foothold inside the Kubernetes cluster
* Execution - Techniques used to execute code on the cluster
* Persistence - Techniques used to maintain long term access to the cluster
* Privilege Escalation - Techniques used to gain a higher level of access within the cluster
* Defense Evasion - Techniques used to avoid detection by security controls
* Credential Access - Techniques used to steal credentials from the cluster
* Discovery - Techniques used to gather information about the cluster
* Lateral Movement - Techniques used to move laterally within the cluster
* Collection - Techniques used to gather data from the cluster
* Impact - Techniques used to disrupt the cluster

![](https://www.microsoft.com/en-us/security/blog/wp-content/uploads/2021/03/Matrix.png)

## Prerequisites

Before you can start the Kubernetes Security village, the following command line interface tools must be installed on your machine.

### AWS Command Line Interface

1. Follow the [Installing or updating to the latest version of the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html){: target="_blank" rel="noopener"} instructions.

1. Verify the AWS CLI is installed correctly by running the following command in your Terminal:

    ```bash
    aws --version
    ```

    !!! abstract "Expected Output"
        ```
        aws-cli/2.2.18 Python/3.12.7 Darwin/23.6.0 source/arm64
        ```

### Kubectl Command Line Interface

1. Follow the [kubectl install tools](https://kubernetes.io/docs/tasks/tools/install-kubectl/){: target="_blank" rel="noopener"} instructions.

1. In your Terminal, run the following commands to verify the command line tools are installed correctly before moving forward:

    ```bash
    kubectl version --client
    ```

    !!! abstract "Expected Output"
        ```
        Client Version: v1.31.2
        Kustomize Version: v5.4.2
        ```

## Getting Started

Start by visiting the Kubernetes Security Village table. Your village hosts, Eric Johnson and Eric Mead, will provide you with a set of *stolen AWS access keys* to simulate a compromise. From there, it is up to you to gain [Initial Access](./initial-access.md) to the Kubernetes cluster and discover the flags hidden in the environment.
