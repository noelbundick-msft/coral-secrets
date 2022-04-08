## Splitting dev/prod secrets

```shell
# Create a new gpg key for encrypting prod secrets
KEY_NAME="github.com/noelbundick-msft/coral-secrets-prod"
gpg --batch --full-generate-key <<EOF
%no-protection
Key-Type: 1
Key-Length: 4096
Subkey-Type: 1
Subkey-Length: 4096
Expire-Date: 0
Name-Comment: prod secrets
Name-Real: ${KEY_NAME}
EOF

# Get the generated key for use with SOPS
gpg --list-secret-keys "${KEY_NAME}"
# Note: This will be different for you, but I don't feel like parsing gpg output right now
# Note this is different from the dev key (7DB6EF589A3D43370E06AF4D13C5845206955A89)
export SOPS_PGP_FP=5F3CFA12F427D24555990C6659D488A7318723E3

# Create a new "prod" cluster
az aks create -g coral-secrets -n fakeprod
az aks get-credentials -g coral-secrets -n fakeprod --admin

# Register the cluster w/ Coral
cat <<EOF > ./clusters/fakeprod.yaml
kind: Cluster
metadata:
  name: fakeprod
  labels:
    type: aks
spec:
  environments:
    - prod
EOF
git add ./clusters/*.yaml
git commit -m "Add a prod cluster"
git push

# Bootstrap the cluster to the gitops repo
export GITHUB_TOKEN=<REDACTED>
flux bootstrap github --personal=true --owner=noelbundick-msft --repository=coral-secrets-gitops --branch=main --path=clusters/fakeprod

# Create a secret in k8s as part of bootstrapping - this will be used to decrypt secrets
# NOTE: this has the same name, but the contents are the prod key, not the first (dev) key
gpg --export-secret-keys --armor "${SOPS_PGP_FP}" |
  kubectl create secret generic sops-gpg \
  --namespace=flux-system \
  --from-file=sops.asc=/dev/stdin

# IMPORTANT: this needs to happen before flux-system tries to use an encrypted secret, otherwise it can never reconcile
# Break into jail: update the GitOps repo directly to tell flux to use the sops decryption provider
pushd $(mktemp -d)
git clone https://github.com/noelbundick-msft/coral-secrets-gitops.git
cd coral-secrets-gitops
cat <<EOF >> ./clusters/fakeprod/flux-system/gotk-sync.yaml
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

# We'll try to use a prod secret pointing to the same registry

# Define a secret, tokenize the namespace, and add it to external-service
## BIG PROBLEM! We can't do secrets in the template, unless we make everyone choose "external-service-dev" and "external-service-prod" templates
## This may not be the absolute worst re: template versioning (still TBD - we should just use Helm or something)
## But it's a different problem from "prod vars should be different than dev vars"
## We also removed the ability for platform teams to override template variables
## We may also want a bunch of different SOPS keys for different use cases? Does a cluster-level solution close any doors for later?
## "Environment" feels like a potential special-case to attach to, but we made the decision to make cluster support multiple environments, which breaks this (spec.decryption.secretRef is single-valued)
## It's not really about what environment the app is - but truly "what encryption key needs to be used to decrypt this?"
## We have a single Flux Kustomization object. This seems okay but broken apps break infra - that's no good. Should we reconsider? Does that change the decryption key solution?

# Lazy attempt 1:
# Define a SOPS_PGP_FP label on each cluster
# If any file names match *.{SOPS_PGP_FP}.yaml, then use that during rendering instead of the normal file
# This could just as easily be a folder named {SOPS_PGP_FP}, and we just take anything in that folder and copy it into the output before committing to the gitops repo
# Thsi would be like a prebaked Kustomization overlay that replaces files - I'm blanking on the correct term here

# Base file (bogus values, not encrypted, just shows what it will look like in the cluster
# Maybe it has nonsecret defaults for clusters that don't have an encryption key (ex: codespace/dev environments)?)
# This sounds like it introduces risk, so we probably don't want that
kubectl create secret docker-registry acr-secret \
  --namespace '%NAMESPACE%' \
  --docker-server=coralsecrets.azurecr.io \
  --docker-username=bogus \
  --docker-password=bogus \
  --dry-run=client -o yaml |
  sops --input-type=yaml --output-type=yaml --encrypt --unencrypted-regex '^(namespace)$' /dev/stdin |
  sed -e "s/'%NAMESPACE%'/{{coral.workspace}}-{{coral.app}}-{{coral.deployment}}/" > ./templates/external-service/deploy/imagePullSecret.yaml

# Prod-encrypted file
kubectl create secret docker-registry acr-secret \
  --namespace '%NAMESPACE%' \
  --docker-server=coralsecrets.azurecr.io \
  --docker-username=$CLIENT_ID \
  --docker-password=$CLIENT_SECRET \
  --dry-run=client -o yaml |
  sops --input-type=yaml --output-type=yaml --encrypt --unencrypted-regex '^(namespace)$' /dev/stdin |
  sed -e "s/'%NAMESPACE%'/{{coral.workspace}}-{{coral.app}}-{{coral.deployment}}/" > ./templates/external-service/deploy/imagePullSecret.${SOPS_PGP_FP}.yaml

# This kind of falls apart due to how coral renders applications - which is once per target
# Fully denormalized would be noisy but would solve the problem here
#  * Added benefit of showing the true blast radius of a proposed change in the git diff very obvious
#  * Remove a level of indirection. Moves towards a cluster being completely self-contained: good for these cloud-baked -> airgapped scenarios
# Writing an optional patch per-cluster would also work - but folks seem to find Kustomize/patches very confusing
# Turns out baking this into the rendered output isn't possible without a design change - render doesn't have access to the Cluster object. This is by design. Prior to now, we had no reason to consider cluster-level variables when writing to the gitops repo. We could probably render on a "per encryption key" basis, which wouldn't be incorrect - but would be confusing imo

# Moderately lazy thought #2:
# Is this a more general "something specific about this cluster/label needs to be changed" problem?
# This might be Pandora's box. Assume we override A for SOPS_GPG_FP label X. And then we also override A based on "cloud: azure". Who wins? We want to make things easier, not harder
# Maybe punt on this until we have a real problem to solve
```
