# Hacking w/ Coral + secrets

In a terminal somewhere

```shell
export GITHUB_TOKEN=<pat w/ [repo,workflow] + SSO to Microsoft>
npx @coraldev/cli@latest init --control-plane noelbundick-msft/coral-secrets
```

Go to https://github.com/noelbundick-msft/coral-secrets and create a Codespace. Inside, do this:

```shell
# Install some tools
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
curl -s https://fluxcd.io/install.sh | sudo bash
sudo chmod +x /usr/local/bin/flux
sudo curl -L https://github.com/mozilla/sops/releases/download/v3.7.2/sops-v3.7.2.linux.amd64 -o /usr/local/bin/sops
sudo chmod +x /usr/local/bin/sops

# Create a new gpg key for encrypting secrets
KEY_NAME="github.com/noelbundick-msft/coral-secrets"
gpg --batch --full-generate-key <<EOF
%no-protection
Key-Type: 1
Key-Length: 4096
Subkey-Type: 1
Subkey-Length: 4096
Expire-Date: 0
Name-Comment: flux secrets
Name-Real: ${KEY_NAME}
EOF

# Get the generated key for use with SOPS
gpg --list-secret-keys "${KEY_NAME}"
# Note: This will be different for you, but I don't feel like parsing gpg output right now
export SOPS_PGP_FP=7DB6EF589A3D43370E06AF4D13C5845206955A89

# Create a cluster
k3d cluster create

# Register the cluster w/ Coral
cat <<EOF > ./clusters/codespace.yaml
kind: Cluster
metadata:
  name: codespace
  labels:
    type: codespace
spec:
  environments:
    - dev
EOF
git commit -m "Add a cluster"
git push

# Bootstrap the cluster to the gitops repo
export GITHUB_TOKEN=<REDACTED>
flux bootstrap github --personal=true --owner=noelbundick-msft --repository=coral-secrets-gitops --branch=main --path=clusters/codespace 

# Create a secret in k8s as part of bootstrapping - this will be used to decrypt secrets
gpg --export-secret-keys --armor "${SOPS_PGP_FP}" |
  kubectl create secret generic sops-gpg \
  --namespace=flux-system \
  --from-file=sops.asc=/dev/stdin

# IMPORTANT: this needs to happen before flux-system tries to use an encrypted secret, otherwise it can never reconcile
# Break into jail: update the GitOps repo directly to tell flux to use the sops decryption provider
pushd $(mktemp -d)
git clone https://github.com/noelbundick-msft/coral-secrets-gitops.git
cd coral-secrets-gitops
cat <<EOF >> ./clusters/codespace/flux-system/gotk-sync.yaml
  decryption:
    provider: sops
    secretRef:
      name: sops-gpg
EOF
git commit -am "Add sops dencryption provider"
git push
popd

# Ensure that Flux accepted the change
flux reconcile kustomization flux-system --with-source

# Create a container registry - happens to be Azure but it's just a generic private registry here
az login
az group create -n coral-secrets -l westus3
az acr create -g coral-secrets -n coralsecrets --sku Standard

# Create a Service Principal to pull containers
SP_CREDS=$(az ad sp create-for-rbac -n noel-coral-secrets)
CLIENT_ID=$(echo $SP_CREDS | jq -r .appId)
CLIENT_SECRET=$(echo $SP_CREDS | jq -r .password)

# Grant the SP the AcrPull role on the registry
ACR_ID=$(az acr show -n coralsecrets --query id -o tsv)
az role assignment create --assignee $CLIENT_ID --scope $ACR_ID --role AcrPull

# Add a sample container
az acr login -n coralsecrets
docker pull nginx
docker tag nginx coralsecrets.azurecr.io/nginx
docker push coralsecrets.azurecr.io/nginx

# Define a secret, tokenize the namespace, and add it to external-service
kubectl create secret docker-registry acr-secret \
  --namespace '%NAMESPACE%' \
  --docker-server=coralsecrets.azurecr.io \
  --docker-username=$CLIENT_ID \
  --docker-password=$CLIENT_SECRET \
  --dry-run=client -o yaml |
  sops --input-type=yaml --output-type=yaml --encrypt --unencrypted-regex '^(namespace)$' /dev/stdin |
  sed -e "s/'%NAMESPACE%'/{{coral.workspace}}-{{coral.app}}-{{coral.deployment}}/" > ./templates/external-service/deploy/imagePullSecret.yaml

# Register the encrypted secret with the template
echo '  - imagePullSecret.yaml' >> ./templates/external-service/deploy/kustomization.yaml

# Add the imagePullSecrets reference to the deployment
cat <<EOF >> ./templates/external-service/deploy/deployment.yaml
      imagePullSecrets:
        - name: acr-secret
EOF

# Commit the new template changes
git add ./templates/external-service-advanced/deploy/*.yaml
git commit -m "Add encrypted imagePullSecret"
git pull --no-rebase
git push

# Onboard a new workspace + default target
cat <<EOF > ./workspaces/noel.yaml
kind: Workspace
metadata:
  name: noel
spec: {}
EOF

mkdir -p ./applications/noel/Targets
cat <<EOF > ./applications/noel/Targets/default.yaml
kind: Target
metadata:
  name: default
spec:
  workspace: noel
  selector: {}
EOF
git add ./workspaces/*.yaml
git add ./applications/noel/Targets/*.yaml
git commit -m "Onboard new workspace"
git push
```

Go create a new app repo. Contents don't matter. Fill in the following `/app.yaml`

```yaml
template: external-service
deployments:
  development:
    target: default
    clusters: 1
    values:
      image: coralsecrets.azurecr.io/nginx
      port: 80
```

Now register it in the control plane

```shell
# Onboard an application
mkdir -p ./applications/noel/ApplicationRegistrations
cat <<EOF > ./applications/noel/ApplicationRegistrations/coral-secrets-app.yaml
kind: ApplicationRegistration
metadata:
  name: coral-secrets-app
spec:
  workspace: noel
  app:
    repo: https://github.com/noelbundick-msft/coral-secrets-app
    ref: main
    path: app.yaml
EOF

git add ./applications/noel/ApplicationRegistrations/*.yaml
git commit -m "Add an application"
git push

# NOTE: the pod is actually running tut will show ready 0/1 because nginx doesn't expose /healthcheck for the readiness probe
kubectl port-forward -n noel-coral-secrets-app-development deploy/coral-secrets-app-deployment 8080:80

# Great success! You pulled from a private registry using an encrypted secret
curl localhost:8080
```
