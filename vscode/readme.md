# VS Code with GPU
This recipe schedules an interactive VS Code session on TIDE.
This is useful when you need to run multiple instances of VS Code or when estimating resource allocations (i.e. CPUs, GPUs, and memory) while developing and testing your code.
A pod is allowed to run for 6 hours, so if you need to run for longer than that consider using a [deployment or job](https://csu-tide.github.io/batch-jobs/#deployments).

A quick note on the choice of storage for this recipe.
For the home directory, we are using Ceph Block storage mounted at the path `/home/jovyan` which is ReadWriteOnce, meaning that you can only attach your home directory to one job at a time.
In addition to the home directory, we could make use of File Storage and/or Object Storage which are ReadWriteMany, meaning that multiple jobs can access the data at the same time.
For more information on storage types, please see the [Storage Services Overview](https://csu-tide.github.io/storage-services/).

A quick note on the choice of image for this recipe.
We are re-using the same Jupyter PyTorch container image available on our JupyterHub instance, so if you are moving to batch jobs from JupyterHub, then you should already be familiar with the OS, utilities and software installed.
This is purely for convenience -- using a Jupyter container image is by no means a requirement.
You should be able to install and run code-server (VS Code) on any container image with sufficient (I.E. root/sudo) privileges.
- *Note*: If you choose a non-Jupyter image option, you may need to modify the path where your home directory mounts (I.E. change it to /root).

## Ingredients
- [Local installation of kubectl](../README.md#install-kubectl)
- (Optional) Preferred container image 

## Prep
1. Set your namespace in an environment variable `ns`:
    - macOS & Linux:
        - `ns=[your-namespace-here]`
    - Windows (PowerShell)
        - `$ns="[your-namespace-here]"`
    - *Note*: Make sure to remove the brackets '[' & ']'
1. (Optional) Browse the [available container images](https://csu-tide.github.io/jupyterhub/images) to see what software you want to run
1. (Optional) Edit the file `vscode-pod.yaml` and replace the value for the `image:` line with the url of your chosen container image

## Instructions

### One-time Steps
The steps in this section should only be followed if you do not already have a home directory in your namespace.
You will typically follow these steps if this is your first time through this recipe.

1. Modify the file `template-pvc.yaml` and prepend your SSO ID (university email) prefix on line 4, in front of "-home"
    - *I.E.*: If your SSO ID is maztec@sdsu.edu, line 4 should be modified to "maztec-home" 
1. Create your home directory:
    ```bash
    kubectl -n $ns apply -f template-pvc.yaml
    ```
    - *Note*: If you get a validation warning message like 'The PersistentVolumeClaim "-home" is invalid...', then please verify that you saved your changes to the file
1. Modify the file `vscode-pod.yaml` to attach your home directory in the spec.volumes.home entry (line 58) such that the value matches "[your-sso-id-prefix]-home"; I.E.: "maztec-home"

### Running VS Code
1. (Optional) If you are running this in a shared namespace with other users, then we highly recommend that you append your SSO ID prefix to the metadata.name value (line 4)
        - *I.E*: If your SSO ID is maztec@sdsu.edu, line 4 should be modified to "vscode-gpu-maztec"
    - *Note*: If you get a validation warning message like 'The Pod "vscode-gpu" is invalid:...', then please verify that you saved your changes to the file
1. Schedule your VS Code job onto TIDE:
    ```bash
    kubectl -n $ns apply -f vscode-pod.yaml
    ```
1. Check the status of your pod:
    ```bash
    kubectl -n $ns get pods
    ```    
    - or
    ```bash
    kubectl -n $ns get pods --watch
    ```
    - *Note*: The `--watch` option must be ended with `ctrl + c`
    - You should see the following:
        ```
        NAME               READY   STATUS    RESTARTS   AGE
        vscode-gpu           1/1     Running   0          8m50s
        ```
    - *Note:* If you modified the pod name in step 1 above, then you should see that value instead
1. Forward port 8080 from the job running on TIDE to your local machine
    - *Note:* If you modified the pod name in step 1 above, then you must use that value for the pod name in the command below; I.E.: vscode-gpu-maztec
    ```bash
    kubectl -n $ns port-forward vscode-gpu 8080
    ```
    - *Note*: This command must be running the entire time that you wish to access VS Code, if you see a connection error message in your browser, check the status of the port-forward command
    - *Note*: VS Code will continue running regardless of running port-forward, meaning that if you disconnect, then you can reconnect at a later time (assuming the job has not been stopped nor reached its maximum runtime)
1. Open your web browser and navigate to [http://127.0.0.1:8080/?folder=/home/jovyan](http://127.0.0.1:8080/?folder=/home/jovyan)
1. Start using VS Code in the browser

## Clean Up
1. Hit `ctrl + c` to stop your port-forward command
1. Stop your VS Code pod: 
    - You may leave the job running until it reaches the max runtime at which point it will be shut down automatically
        - Or
    - You can tell k8s to shut it down manually with this command:
        - `kubectl -n $ns delete -f vscode-pod.yaml`
