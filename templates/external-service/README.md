# External Service

This template exposes a minimal set of parameters needed for App Teams to deploy an externally available web application.

## Configuration

Applications will be deployed with host-based routing that includes the workspace, application name, and deployment name. Ex: `dev.myapp.workspaceA.example.com`

The default domain suffix is `example.com` and can be changed by modifying [`template.yaml`](./template.yaml)

## Values

- image - The image specification including tag of the container image to use.
- port - The port on the container where the app can be accessed
- domainSuffix (optional): top level domain that will be used for Ingress resources
