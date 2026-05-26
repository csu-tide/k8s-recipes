# GitHub Private Repository Clone
This recipe schedules a pod that clones a private GitHub repository into the pod before the main container starts.
It uses the [git-sync](https://github.com/kubernetes/git-sync) init container to clone the repository one time with a GitHub fine-grained personal access token (PAT).

## Ingredients
- [Local installation of kubectl](../README.md#install-kubectl)
- GitHub account with access to the private repository
- Private repository HTTPS URL
- Fine-grained GitHub PAT with read-only access to the repository contents

## Prep
1. Open your terminal
1. Set your namespace in an environment variable `ns`:
    - macOS & Linux:
        - `ns=[your-namespace-here]`
    - Windows (PowerShell)
        - `$ns="[your-namespace-here]"`
    - *Note*: Make sure to remove the brackets '[' & ']'
1. Find your private repository HTTPS URL:
    - Example: `https://github.com/my-github-user/my-private-repo.git`
1. Choose a Kubernetes Secret name for your PAT:
    - This recipe uses `git-password`
    - *Note*: If you are in a shared namespace, use a unique secret name such as `git-password-maztec`

## Instructions

### Create a Fine-Grained PAT
1. In GitHub, click your profile picture and open **Settings**
1. In the left sidebar, open **Developer settings**
1. Open **Personal access tokens** and choose **Fine-grained tokens**
1. Click **Generate new token**
1. Enter a descriptive name, such as `tide-git-sync-readonly`
1. Set an expiration date
1. Under **Repository access**, choose **Only select repositories**
1. Select the private repository that you want to clone
1. Under **Repository permissions**, set **Contents** to **Read-only**
1. Leave other permissions unset unless your repository owner requires them
1. Click **Generate token**
1. Copy the token value before leaving the page
    - *Note*: GitHub only shows the token value once
    - *Note*: Organization repositories may require owner approval before the token can access the repository

### Add the PAT to Your Namespace
1. Create a Secret containing your PAT:
    ```bash
    kubectl -n $ns create secret generic git-password --from-literal=git-password='YOUR_PAT_VALUE'
    ```
    - *Note*: Replace `YOUR_PAT_VALUE` with the token value copied from GitHub
    - *Note*: If you chose a unique Secret name, replace the first `git-password` with that name
1. Verify that the Secret exists:
    ```bash
    kubectl -n $ns get secret git-password
    ```

### Update the Pod Manifest
1. Modify the file `git-sync-pod.yaml`
1. Replace the `--repo` value with your private repository HTTPS URL:
    ```yaml
    - --repo=https://github.com/my-github-user/my-private-repo.git
    ```
1. Replace the `--username` value with your GitHub username:
    ```yaml
    - --username=my-github-user
    ```
1. If you changed the Secret name, update the `secretKeyRef.name` value:
    ```yaml
    name: git-password-maztec
    ```
1. Save the file

### Clone and Access the Repository
1. Schedule the pod:
    ```bash
    kubectl -n $ns apply -f git-sync-pod.yaml
    ```
    - You should see the following:
        ```
        pod/git-sync-example created
        ```
1. Watch the pod until it is ready:
    ```bash
    kubectl -n $ns get pods --watch
    ```
    - *Note*: Hit `ctrl + c` when the READY column shows `1/1`
1. If the pod does not start, check the git-sync init container logs:
    ```bash
    kubectl -n $ns logs git-sync-example -c git-sync
    ```
1. Open a shell in the pod:
    ```bash
    kubectl -n $ns exec -it git-sync-example -- /bin/bash
    ```
1. List the cloned repository:
    ```bash
    ls -la /home/jovyan/src/current
    ```
    - *Note*: The `current` directory is a link created by git-sync that points to the cloned repository contents

## Clean Up
1. Delete the pod:
    ```bash
    kubectl -n $ns delete -f git-sync-pod.yaml
    ```
    - You should see the following:
        ```
        pod "git-sync-example" deleted
        ```
1. Delete the Secret if you no longer need it:
    ```bash
    kubectl -n $ns delete secret git-password
    ```
1. If the PAT is no longer needed, revoke it in GitHub:
    - GitHub **Settings** > **Developer settings** > **Personal access tokens** > **Fine-grained tokens**
