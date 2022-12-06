# kpack image registry 

This tutorial integrates `kpack` and `cosign` while using `dockerhub` image registry as source code repository.
Instead of using git, the application source code is packaged using `imgpkg`, and pushed as an OCI image to `dockerhub`.
- **source code image**: 
  - name: `guybalmas/src-code-image:latest`
  - url: `https://hub.docker.com/repository/docker/guybalmas/src-code-image`
  - 
## Prerequisites
1. Install `kpack` CLI:
   - https://github.com/vmware-tanzu/kpack-cli
```bash
wget "https://github.com/vmware-tanzu/kpack-cli/releases/download/v0.8.0/kp-linux-amd64-0.8.0"
mv kp-linux-amd64-0.8.0 /usr/local/bin/kp
chmod +x /usr/local/bin/kp
```
2. Install `cosign` CLI:
   - https://docs.sigstore.dev/cosign/installation/
```bash
wget "https://github.com/sigstore/cosign/releases/download/v1.6.0/cosign-linux-amd64"
mv cosign-linux-amd64 /usr/local/bin/cosign
chmod +x /usr/local/bin/cosign
```
3. Install `imgpkg` CLI:
    - https://github.com/vmware-tanzu/carvel-imgpkg/releases
```bash
wget "https://github.com/vmware-tanzu/carvel-imgpkg/releases/download/v0.34.1/imgpkg-linux-amd64"
mv imgpkg-linux-amd64 /usr/local/bin/imgpkg
chmod +x /usr/local/bin/imgpkg
```

## Solution
### Step 1 - Install kpack onto the cluster
```bash
# Will be applied to kpack namespace by default
kubectl apply -f https://github.com/pivotal/kpack/releases/download/v0.8.2/release-0.8.2.yaml

# Make sure all pods are running and the enviroment is ready
kubectl get pods --namespace kpack --watch
```

### Step 2 - Apply kpack CRD's onto the cluster
```bash
# Export dockerhub credentials as env vars
export DOCKERHUB_USER=<username>
export DOCKERHUB_PASSWORD=<password>

# Create dockerhub credentials secret
kubectl create secret docker-registry docker-registry-credentials \
    --docker-username=$DOCKERHUB_USER \
    --docker-password=$DOCKERHUB_PASSWORD \
    --docker-server=https://index.docker.io/v1/ \
    --namespace bundled-apps --dry-run=client -o yaml > kpack/docker-registry-credentials.yaml

# Apply by order
kubectl apply -f kpack/ns.yaml
kubectl apply -f kpack/sa.yaml
kubectl apply -f kpack/docker-registry-credentials.yaml
kubectl apply -f kpack/cluster-store.yaml
kubectl apply -f kpack/cluster-stack.yaml
kubectl apply -f kpack/cluster-builder.yaml
kubectl apply -f kpack/image.yaml

# Check resources
kp clusterstore status my-cluster-store 
kp clusterstack status base
kp clusterbuilder status my-cluster-builder

# View image resource status
kubectl get images -n bundled-apps
kubectl get image my-bundle-image -n bundled-apps
kp image status my-bundle-image -n bundled-apps

# View build logs of my-image
kp build logs my-bundle-image -n bundled-apps

#trigger by hand if needed
kp image trigger my-bundle-image -n bundled-apps
```

### Step 3 - Cosign integration
```bash
# Generate a key-pair secret
# command: cosign generate-key-pair k8s://<namespace>/<key-pair-name>
cosign generate-key-pair k8s://bundled-apps/cosign-key-pair
kubectl get secret cosign-key-pair -n bundled-apps -o yaml > kpack/cosign-secret.yaml
```
1. Edit cosign-secret's manifest
```bash
vim kpack/cosign-secret.yaml
```
2. Add annotation to `kpack/cosign-secret.yaml`:

If you are using `dockerhub` or a registry that does not support OCI media types, you need to add the annotation `kpack.io/cosign.docker-media-types: "1"` to the cosign secret. 
```yaml
metadata:
   annotations:
      # add this annotation
      kpack.io/cosign.docker-media-types: "1"
```
The secret `cosign-key-pair` should look something like this:
```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: cosign-key-pair
  namespace: bundled-apps
  annotations:
    kpack.io/cosign.docker-media-types: "1"
data:
  cosign.key: <PRIVATE KEY DATA>
  cosign.password: <COSIGN PASSWORD>
  cosign.pub: <PUBLIC KEY DATA>
```

3. Apply onto the cluster:
```bash
kubectl apply -f cosign-secret.yaml

#validate
kubectl describe secret -n bundled-apps cosign-key-pair
```
**output:**
```bash
Name:         cosign-key-pair
Namespace:    bundled-apps
Labels:       <none>
Annotations:  kpack.io/cosign.docker-media-types: 1

Type:  Opaque

Data
====
cosign.key:       649 bytes
cosign.password:  3 bytes
cosign.pub:       178 bytes
```

4. Configure the service-account for access to cosign key-pair:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
   name: my-service-account
   namespace: bundled-apps
secrets:
   - name: docker-registry-credentials
   - name: cosign-key-pair
imagePullSecrets:
   - name: docker-registry-credentials
```
5. Apply onto the cluster:
```bash
kubectl apply -f kpack/sa.yaml
```

### Step 4 - Verify
```bash
# get the latest image with digest:
kubectl get image my-bundle-image -n bundled-apps

cosign verify --key cosign.pub <latest-image-with-digest>

#for example:
cosign verify --key cosign.pub guybalmas/example-app-oci@sha256:8a10e2e4a0477e22be260e61ac136d9f0ea0a6d5649dd1d03d37b54c56b1b0c9
```
**output**
```json
{
   "critical":{
      "identity":{
         "docker-reference":"index.docker.io/guybalmas/example-app-oci"
      },
      "image":{
         "docker-manifest-digest":"sha256:8a10e2e4a0477e22be260e61ac136d9f0ea0a6d5649dd1d03d37b54c56b1b0c9"
      },
      "type":"cosign container image signature"
   },
   "optional":{
      "buildNumber":"1",
      "buildTimestamp":"20221206.084838"
   }
}
```

### Step 5 - Trigger builds
When monitoring an image registry as source registry, `kpack` doesn't rebuild when `latest` image changes.
We will need to trigger `kpack` actively whenever a new source image is pushed to the image registry.
```bash
# Push packaged source code image to dockerhub
imgpkg push -i guybalmas/src-code-image -f .
```
**output:**
```bash
dir: .
file: .gitignore
dir: .idea
file: .idea/.gitignore
file: .idea/compiler.xml
file: .idea/encodings.xml
file: .idea/jarRepositories.xml
file: .idea/misc.xml
file: .idea/vcs.xml
file: .idea/workspace.xml
file: README.md
dir: config
file: config/config.yml
file: config/values.yml
file: pom.xml
dir: src
dir: src/main
dir: src/main/java
dir: src/main/java/org
dir: src/main/java/org/example
dir: src/main/java/org/example/app
file: src/main/java/org/example/app/HelloController.java
file: src/main/java/org/example/app/exampleapp.java
dir: src/main/resources
file: src/main/resources/application.properties
dir: src/test
dir: src/test/java
dir: src/test/java/org
dir: src/test/java/org/example
dir: src/test/java/org/example/app
file: src/test/java/org/example/app/exampleappTests.java
Pushed 'index.docker.io/guybalmas/src-code-image@sha256:9cd8b37a405600632bd969f9d216f6b302ea63a68dd69243742ed0c4468ea61f'
Succeeded
```
You can verify the image pushed by `imgpkg`, by applying it onto the cluster:
```bash
# Optional: verify the image 
mkdir ~/temp
imgpkg pull -i index.docker.io/guybalmas/src-code-image@sha256:<sha256> -o ~/temp
cd ~/temp

# Build and deploy
mkdir .imgpkg 
kbld -f config/ --imgpkg-lock-output .imgpkg/images.yml
kbld -f config/ -f .imgpkg/images.yml | kubectl apply -f-
```
**output:**
```bash
resolve | final: guybalmas/example-app-oci -> index.docker.io/guybalmas/example-app-oci@sha256:8a10e2e4a0477e22be260e61ac136d9f0ea0a6d5649dd1d03d37b54c5
6b1b0c9
service/example-app created
deployment.apps/bundle-app created
```
Now that we have a new image in the registry, we can trigger `kpack` to build the OCI image:
```bash
# Trigger by hand
kp image trigger my-bundle-image -n bundled-apps

# View current builds 
kp build list -n bundled-apps

# View image resource status
kubectl get images -n bundled-apps
kubectl get image my-bundle-image -n bundled-apps
kp image status my-bundle-image -n bundled-apps

# View build logs of my-image
kp build logs my-bundle-image -n bundled-apps
```

