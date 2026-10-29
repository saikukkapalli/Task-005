# Kubernetes GitOps and Automated Image Updates Using FluxCD

A simple GitOps project that automatically builds a website image,
publishes it to Docker Hub, detects the new image with FluxCD, updates
the image tag in Git, and deploys the new version to a local Kubernetes
cluster.

## Project overview

### Goal

Demonstrate the full image automation workflow:

1.  Edit the website source in `app/index.html`.
2.  Push the change to the `main` branch on GitHub.
3.  GitHub Actions builds a Docker image and pushes it to Docker Hub
    with a numeric tag.
4.  FluxCD scans Docker Hub and selects the highest numeric image tag.
5.  Flux image automation updates `manifests/deployment.yml` and commits
    the change to GitHub.
6.  Flux's Git reconciliation applies the updated manifest to
    Kubernetes.
7.  Kubernetes automatically rolls out a new Pod using the updated
    image.

> The build number/tag is not automatically shown inside the HTML unless
> the workflow and Dockerfile explicitly inject it. The image tag and
> the text displayed on the website are separate things.

## Architecture

``` text
Developer edits app/index.html
             |
             v
       Push to GitHub
             |
             v
   GitHub Actions workflow
   builds and pushes image
             |
             v
          Docker Hub
             |
             v
 Flux Image Reflector Controller
      ImageRepository scan
             |
             v
         ImagePolicy
   selects highest numeric tag
             |
             v
 Flux Image Automation Controller
 updates manifests/deployment.yml
       and commits to GitHub
             |
             v
     Flux GitRepository
             |
             v
      Flux Kustomization
 applies the updated manifest
             |
             v
 Kubernetes Deployment rollout
             |
             v
       Updated website Pod
```

## Repository layout

The project uses the following files and folders. Keep paths consistent
with your actual repository.

``` text
Task-005/
├── app/
│   ├── Dockerfile
│   └── index.html
├── .github/
│   └── workflows/
│       └── build-push.yml
├── manifests/
│   ├── namespace.yml
│   ├── deployment.yml
│   ├── service.yml
│   └── kustomization.yml
└── flux-config/
    └── flux-system/
        ├── gotk-components.yaml
        ├── gotk-sync.yaml
        ├── app-sync.yml
        ├── image-repository.yml
        ├── image-policy.yml
        ├── image-update-automation.yml
        └── kustomization.yaml
```

Flux-generated files may vary depending on how Flux was installed. This
list reflects the files used by this project.

## Technologies

-   **Minikube:** Runs the local Kubernetes cluster.
-   **Docker:** Packages the static website into an image.
-   **Docker Hub:** Stores published image tags.
-   **GitHub Actions:** Builds and pushes a new image when a change is
    pushed to `main`.
-   **FluxCD source-controller:** Fetches the Git repository source.
-   **Flux image-reflector-controller:** Scans the image registry and
    records available tags.
-   **Flux image-automation-controller:** Updates image references in
    Git and pushes commits.
-   **Flux Kustomization:** Applies the manifests from Git to the
    cluster.
-   **Kubernetes Deployment:** Maintains the desired number of
    application Pods and automatically rolls out a changed Pod template.
-   **Kubernetes Service:** Provides a stable in-cluster endpoint for
    the website Pods.

## Configuration notes

### 1. Application Dockerfile

The Dockerfile is in `app/Dockerfile`, so the GitHub Actions build
context must be `./app`.

A simple Dockerfile is:

``` dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

Nginx serves the page on container port 80. `EXPOSE 80` documents the
intended port; it does not publish the port on the host.

### 2. GitHub Actions

The workflow is `.github/workflows/build-push.yml`. It checks out the
repository, logs in to Docker Hub using GitHub Actions secrets, builds
the image from `app/`, and pushes a numeric tag.

The workflow should use these repository secrets:

-   `DOCKERHUB_USERNAME`: Docker Hub username.
-   `DOCKERHUB_TOKEN`: Docker Hub access token with Read & Write
    permission.

Never commit access tokens or private keys to the repository.

The build step should use the application folder:

``` yaml
with:
  context: ./app
  file: ./app/Dockerfile
  push: true
  tags: saicowdary9/flux-cd:${{ github.run_number }}
```

Replace `saicowdary9/flux-cd` if the actual Docker Hub image name
differs.

To avoid an automation loop, trigger the workflow only for application
changes, for example:

``` yaml
on:
  push:
    branches:
      - main
    paths:
      - 'app/**'
```

This prevents Flux commits to `manifests/deployment.yml` from triggering
another image build. If you need workflow-file changes to trigger a run,
add `.github/workflows/build-push.yml` to the `paths` list.

### 3. ImageRepository

The `ImageRepository` tells Flux which registry image to scan and how
often to scan it. In this project, it points to `saicowdary9/flux-cd`
and scans at a one-minute interval.

Check it with:

``` powershell
kubectl get imagerepositories -n flux-system
kubectl describe imagerepository flux-cd-image -n flux-system
```

### 4. ImagePolicy

The `ImagePolicy` references the `ImageRepository` and uses numeric
ordering to select the highest valid numeric tag.

Example policy:

``` yaml
apiVersion: image.toolkit.fluxcd.io/v1
kind: ImagePolicy
metadata:
  name: flux-cd-policy
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: flux-cd-image
  policy:
    numerical:
      order: asc
```

Use numeric tags such as `1`, `2`, `3`, and so on. Tags such as `v1` are
not valid numeric values for this policy.

Check the selected tag with:

``` powershell
kubectl get imagepolicies -n flux-system
kubectl describe imagepolicy flux-cd-policy -n flux-system
```

### 5. Image marker in the Deployment

The image line in `manifests/deployment.yml` must contain the setter
marker that matches the policy name:

``` yaml
image: saicowdary9/flux-cd:3 # {"$imagepolicy": "flux-system:flux-cd-policy"}
```

The tag shown above is an example. Flux Image Automation updates the tag
when a newer tag is selected.

### 6. ImageUpdateAutomation

The `ImageUpdateAutomation` watches the configured Git source, updates
manifests under `./manifests`, and pushes the changes to the `main`
branch. Its Git credentials must permit writing to the repository.

Check it with:

``` powershell
kubectl get imageupdateautomations -n flux-system
kubectl describe imageupdateautomation flux-cd-image-automation -n flux-system
```

A `READY=True` status means the resource is currently healthy; inspect
the status message and recent commits as well to verify the expected
image update occurred.

### 7. GitRepository and Kustomization

-   **GitRepository** fetches the configured Git repository and branch.
-   **Kustomization** applies the manifests from the repository to the
    cluster at its reconciliation interval.

Useful checks:

``` powershell
kubectl get gitrepositories -A
kubectl get kustomizations -A
kubectl describe kustomization flux-cd-app -n flux-system
```

The Kustomization name/namespace can differ in your installation. Use
the name shown by `kubectl get kustomizations -A`.

## Setup and demonstration

### Prerequisites

-   Docker Desktop
-   Minikube and `kubectl`
-   Flux CLI
-   A GitHub repository
-   A Docker Hub repository
-   GitHub Actions secrets configured for Docker Hub

### Start the local cluster

If the Minikube cluster is not already running:

``` powershell
minikube start
kubectl get nodes
```

### Make a website change

Edit `app/index.html`, for example:

``` html
<!DOCTYPE html>
<html>
<head>
  <title>Flux CD Demo</title>
</head>
<body>
  <h1>Hello from Flux CD!</h1>
  <p>Automated GitOps deployment test</p>
</body>
</html>
```

Commit and push the change:

``` powershell
git add app/index.html
git commit -m "Update website content"
git pull --rebase origin main
git push origin main
```

If the rebase reports a conflict, resolve it carefully before pushing.
Do not force-push over Flux's commits.

### Follow the pipeline

1.  Open the repository's **Actions** tab and confirm the build
    succeeds.
2.  Check Docker Hub for the new numeric tag.
3.  Check the Flux image policy for the selected tag.
4.  Check GitHub `manifests/deployment.yml` for Flux's new image tag
    commit.
5.  Wait for Flux Kustomization reconciliation and the Kubernetes
    rollout.
6.  Open the local website and confirm the updated HTML.

Useful commands:

``` powershell
kubectl get imagerepositories -n flux-system
kubectl get imagepolicies -n flux-system
kubectl get imageupdateautomations -n flux-system
kubectl get kustomizations -A
kubectl get deployment flux-cd-app -n flux-cd
kubectl get pods -n flux-cd
kubectl rollout status deployment/flux-cd-app -n flux-cd
kubectl exec -n flux-cd deploy/flux-cd-app -- cat /usr/share/nginx/html/index.html
```

Kubernetes automatically starts a rollout when Flux applies a Deployment
Pod-template change, such as a new image tag. `kubectl rollout status`
only observes the rollout; it does not start it.

## Access the website

For this local demo, port-forward the Kubernetes Service to host port
80:

``` powershell
kubectl port-forward svc/flux-cd-service 80:80 -n flux-cd
```

Keep this terminal open and browse to:

``` text
http://localhost
```

The port-forward process can disconnect if the Pod it is forwarding to
is replaced during a rollout. If that happens, rerun the port-forward
command after the rollout. This is a local access connection issue, not
necessarily a failed deployment.

If port 80 is unavailable, use another local port, such as:

``` powershell
kubectl port-forward svc/flux-cd-service 8082:80 -n flux-cd
```

Then open `http://localhost:8082`.

## Troubleshooting

### Git push is rejected with "fetch first"

Flux may have committed an updated manifest to the remote branch while
you were working locally. Rebase before pushing:

``` powershell
git pull --rebase origin main
git push origin main
```

Do not use `git push --force` to resolve this; it may overwrite Flux's
commits.

### Image automation says the SSH key is read-only

Check the repository's **Settings → Deploy keys** page and confirm the
key used by Flux has write access. The Kubernetes secret referenced by
the GitRepository must contain the matching private key, public key, and
`known_hosts` data in Flux's expected format. After updating
credentials, inspect:

``` powershell
kubectl get gitrepository flux-system -n flux-system
kubectl describe imageupdateautomation flux-cd-image-automation -n flux-system
```

Do not share or commit private keys or secret backup files.

### ImagePolicy does not select the new tag

Check that the new image was pushed to the same repository configured in
`ImageRepository`. Numeric policies require numeric tags; use tags such
as `4` rather than `v4`.

``` powershell
kubectl get imagerepositories -n flux-system
kubectl get imagepolicies -n flux-system
```

### GitHub Actions cannot find Dockerfile

The Dockerfile and HTML are in `app/`, so the build context must be
`./app` and the Dockerfile path must be `./app/Dockerfile`.

### Image tag changed in Git, but Deployment did not update

Check the GitRepository and Kustomization health, then inspect the
Deployment image:

``` powershell
kubectl get gitrepositories -A
kubectl get kustomizations -A
kubectl get deployment flux-cd-app -n flux-cd -o jsonpath="{.spec.template.spec.containers[0].image}"
kubectl describe deployment flux-cd-app -n flux-cd
```

Confirm that the Kustomization points to the correct manifests path and
namespace.

### Deployment updated, but the website shows old HTML

First inspect the HTML inside the running Pod:

``` powershell
kubectl exec -n flux-cd deploy/flux-cd-app -- cat /usr/share/nginx/html/index.html
```

Then compare it with the response from the port-forwarded site. If the
Pod contains the new content but the browser shows old content, check
that you are opening the correct host port and press `Ctrl+F5`. Also
check for an old standalone Docker container publishing a port on the
host:

``` powershell
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Ports}}\t{{.Image}}"
```

### Port-forward disconnects during rollout

A port-forward session can be lost when the selected Pod is replaced.
After the rollout completes, restart:

``` powershell
kubectl port-forward svc/flux-cd-service 80:80 -n flux-cd
```

Keep the terminal open while browsing to `http://localhost`.

### Check general Flux status

``` powershell
flux get sources git -A
flux get kustomizations -A
flux get images all -A
```

If a resource is not ready, inspect its status and events with
`kubectl describe` before changing configuration.

## Security notes

-   Store Docker Hub credentials in GitHub Actions repository secrets.
-   Use a Docker Hub token with only the permissions required to push
    the image.
-   Use a GitHub deploy key with write access only where Flux Image
    Automation needs to push commits.
-   Never commit private keys, access tokens, or secret backups.
-   Keep image tags numeric while using the numerical ImagePolicy.

## Definition of done

-   [ ] Minikube is running and FluxCD controllers are healthy.
-   [ ] Flux is connected to the GitHub repository.
-   [ ] The website is deployed using Flux-managed Kubernetes manifests.
-   [ ] Image Reflector and Image Automation controllers are installed
    and healthy.
-   [ ] GitHub Actions builds and pushes a new image after an
    application change is pushed.
-   [ ] Flux detects the new numeric image tag.
-   [ ] Flux commits the updated image tag to
    `manifests/deployment.yml`.
-   [ ] Flux applies the manifest and Kubernetes rolls out the new image
    automatically.
-   [ ] The updated website content is visible.
-   [ ] This README documents the architecture, configuration, and
    troubleshooting.
