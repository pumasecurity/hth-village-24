# API Key Challenge

The *API Key Challenge* challenge requires you to find a flag stored as a [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/){: target="_blank" rel="noopener"}. Unfortunately, the *kubeace-maverick* IAM user does not have permissions to list secrets. Without this permission, you will need to find the pod using the secret, identify the *secret name*, and access the secret directly.

## Kubernetes Secret Configuration

Kubernetes secrets are often used to store sensitive information, such as passwords, API keys, and private keys, and feed those secrets into a pod as an environment variable or a volume mount. Kubernetes secrets are defined for a pod using the container specification's [volume](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/#create-a-pod-that-has-access-to-the-secret-data-through-a-volume){: target="_blank" rel="noopener"} or an  [environment variable](https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets-as-environment-variables){: target="_blank" rel="noopener"}.

To exfiltrate the secret, you will need to find the name of the secret first. Review the pod specifications in the `hth` namespace. Which pod is referencing a Kubernetes secret? What is the name of the secret?

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
    
    - Use the `kubectl describe pod` command to view each pod's configuration. Review the *Volumes* and *Environment* configurations to identify any secrets being used. Observe that one pod is referencing a Kubernetes secret in an environment variable called *AVIATA_API_KEY*.

        ```bash
        kubectl describe pod -n hth ENTER_POD_NAME
        ```

        !!! abstract "Expected Output"
            ```
            Environment:
              AVIATA_API_KEY:  <set to the key 'value' in secret '?????'>  Optional: false
            ```

    - Note the name of the secret referenced in the pod's environment variable. You will need the name to exfiltrate the flag.

??? success "Answer"

    - The `ui-random-id` pod is referencing a Kubernetes secret named `ui-api-key` in an environment variable called *AVIATA_API_KEY*.

        !!! abstract "Expected Output"
            ```plaintext
            Environment:
              AVIATA_API_KEY:  <set to the key 'value' in secret 'ui-api-key'>  Optional: false
            ```

## Kubernetes Secret Exfiltration

Now that you have identified the Kubernetes secret name, use `kubectl` read the Kubernetes **API Key** secret and decode the flag.

??? tip "Hint"

    - Use the `kubectl get secret` command to read the secret. Observe the output confirms that the secret exists, but does not display the secret's value.

        ```bash
        kubectl get secret -n hth ?????
        ```

        !!! abstract "Expected Output"
            ```
            NAME         TYPE     DATA   AGE
            ?????        Opaque   1      4d18h
            ```

    - Run the Use `kubectl get secret` command again using the output (-o) option to format the response as YAML or JSON. This will display the secret's value in base64 encoding.

        ```bash
        kubectl get secret -n hth ????? -o json
        ```

        !!! abstract "Expected Output"
            ```json hl_lines="4"
            {
              "apiVersion": "v1",
              "data": {
                  "value": "?????"
              },
              "kind": "Secret",
              "metadata": {
                  "creationTimestamp": "2024-11-08T23:19:54Z",
                  "name": "?????",
                  "namespace": "hth",
                  "resourceVersion": "6084",
                  "uid": "84fb5c38-1604-4bb3-a06f-0599b7f832d4"
              },
              "type": "Opaque"
            }
            ```

    - Base64 decode the secret's `value` to reveal the flag.

        ```bash
        echo "?????" | base64 -d
        ```

??? success "Answer"

    Run the following command to decode the secret's value and reveal the flag.

    ```bash
    kubectl get secret -n hth ui-api-key -o json | jq -r .data.value | base64 -d
    ```

    !!! abstract "Expected Output"
        ```plaintext
        hth{?????}
        ```

## Next Challenge

Congratulations! You have successfully located the *API Key* Kubernetes secret being used by the UI pod. Then, decoded the value to reveal the flag.

Continue to the [Cascadia Cockpit Voice Recorders (CVR) Challenge](./cascadia-cvr.md) to learn how the Kubernetes node is authenticating to the private container registry and pulling images.
