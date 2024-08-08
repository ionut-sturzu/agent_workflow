# Agent workflow deployment

## Getting started

This stack deploys an OKE cluster with two nodepools:
- one nodepool with flexible shapes
- one nodepool with GPU shapes

And several supporting applications using helm:
- Nginx
- Cert-manager
- Jupyterhub
- Llama3 model
- CodeLlama model
- CuOpt model

With the scope of demonstrating [nVidia NIM LLM](https://docs.nvidia.com/nim/large-language-models/latest/introduction.html) self-hosted model capabilities in an agent workflow scenario on OCI.

## Prerequisites

**Note:** For helm deployments it's necessary to create bastion and operator host (with the associated policy for the operator to manage the clsuter), **or** configure a cluster with public API endpoint.

In case the bastion and operator hosts are not created, is a prerequisite to have the following tools already installed and configured:
- bash
- helm
- jq
- kubectl
- oci-cli

**NOTE**
In this version it requires **terraform1.2** version as it is specified in the **provider.tf** file

## Helm Deployments

### Nginx

[Nginx](https://kubernetes.github.io/ingress-nginx/deploy/) is deployed and configured as default ingress controller.

### Cert-manager

[Cert-manager](https://cert-manager.io/docs/) is deployed to handle the configuration of TLS certificate for the configured ingress resources. Currently it's using the [staging Let's Encrypt endpoint](https://letsencrypt.org/docs/staging-environment/).

### Jupyterhub

[Jupyterhub](https://jupyterhub.readthedocs.io/en/stable/) will be accessible to the address: [https://jupyter.a.b.c.d.nip.io](https://jupyter.a.b.c.d.nip.io), where a.b.c.d is the public IP address of the load balancer associated with the NGINX ingress controller.

JupyterHub is using a dummy authentication scheme (user/password) and the access is secured using the variables:

```
jupyter_admin_user
jupyter_admin_password
```

It also supports the option to automatically clone a git repo when user is connecting and making it available under `examples` directory.

### Workflow playbook
It will automatically clone the Agent Workflow[https://github.com/ionut-sturzu/notebooks.git] jupiter notebook.

Access the link from terraform output and login with your user name and password.
In the example directory you will find the notebook. First run install the packages by running the first cell.
Then you can run the second cell which will create a gradio interface.
In the Gradio interface put your api keys to secure the access to the model over the internet and click submit. 
At this point the clients are initialized. Now, change tab to Agent workflow and start asking questions.


If you are looking to integrate JupyterHub with an Identity Provider, please take a look at the options available here: https://oauthenticator.readthedocs.io/en/latest/tutorials/provider-specific-setup/index.html

For integration with your OCI tenancy IDCS domain, you may go through the following steps:

1. Setup a new **Application** in IDCS

- Navigate to the following address: https://cloud.oracle.com/identity/domains/

- Click on the `OracleIdentityCloudService` domain

- Navigate to `Integrated applications` from the left-side menu

- Click **Add application**

- Select *Confidential Application* and click **Launch worflow**

2. Application configuration

- Under *Add application details* configure

    name: `Jupyterhub`

    (all the other fields are optional, you may leave them empty)

- Under *Configure OAuth*

    Resource server configuration -> *Skip for later*

    Client configuration -> *Configure this application as a client now*

    Authorization:
    - Check the `Authorization code` check-box
    - Leave the other check-boxes unchecked

    Redirect URL:

    `https://<jupyterhub-domain>/hub/oauth_callback`

- Under *Configure policy*
    
    Web tier policy -> *Skip for later*

- Click **Finish**

- Scroll down wehere you fill find the *General Information* section.

- Copy the `Client ID` and `Client secret`:

- Click **Activate** button at the top.

3. Connect to the OKE cluster and update the JupyterHub Helm deployment values.

- Create a file named `oauth2-values.yaml` with the following content (make sure to fill-in the values relevant for your setup)

    ```yaml
    hub:
    config:
      Authenticator:
        allow_all: true
      GenericOAuthenticator:
        client_id: <client-id>
        client_secret: <client-secret>

        authorize_url:  <idcs-stripe-url>/oauth2/v1/authorize
        token_url:  <idcs-stripe-url>/oauth2/v1/token
        userdata_url:  <idcs-stripe-url>/oauth2/v1/userinfo

        scope:
        - openid
        - email
        username_claim: "email"
      JupyterHub:
        authenticator_class: generic-oauth
    ```

    **Note:** IDCS stripe URL can be fetched from the OracleIdentityCloudService IDCS Domain Overview -> Domain Information -> Domain URL.

    Should be something like this: `https://idcs-18bb6a27b33d416fb083d27a9bcede3b.identity.oraclecloud.com`


- Execute the following command to update the JupyterHub Helm deployment:

    ```bash
    helm upgrade jupyterhub jupyterhub --repo https://hub.jupyter.org/helm-chart/ --reuse-values -f oauth2-values.yaml
    ```



### NIM module

The LLM is deployed using [NIM](https://docs.nvidia.com/nim/index.html).

Parameters:
- `nim_image_repository` and `nim_image_tag` - used to specify the container image location
- `NGC_API_KEY` - required to authenticate with NGC services

### CuOpt Module
The LLM is deployed using [NIM helm chart](https://docs.nvidia.com/cuopt/user-guide/sh-server-overview.html#step-7-installing-cuopt-using-a-helm-chart)

Parameters:
- cuopt_version - link to the helm chart from NGC Catalog
- NGC_API_KEY - same as for the NIM module

The link generated by NGC catalog looks like this: **helm fetch https://helm.ngc.nvidia.com/nvidia/cuopt/charts/cuopt-24.03.00.tgz --username='$oauthtoken' --password=<YOUR API KEY>** and in the code you will need to pass only **https://helm.ngc.nvidia.com/nvidia/charts/cuopt-24.03.00.tgz**

### Codellama
- HF_TOKEN - hugging face token used for login on hugging face
- LLM_API_KEY - an api key to secure the model access over internet
- model_name - name of the codellama model like: **"meta-llama/CodeLlama-7b-Instruct-hf"**

## How to deploy?

1. Deploy via ORM
- Create a new stack
- Upload the TF configuration files
- Configure the variables
- Apply

2. Local deployment

- Change the name of `terraform.auto.tfvars.example` with and add the **NGC_API_KEY**, **HF_TOKEN**, **LLM_API_KEY**. Also, if you want to customize the values of the other variables here is the place.



- Execute the commands

```
terraform init
terraform plan
terraform apply
```

## Known Issues

If `terraform destroy` fails, manually remove the LoadBalancer resource configured for the Nginx Ingress Controller.

After `terrafrom destroy`, the block volumes corresponding to the PVCs used by the applications in the cluster won't be removed. You have to manually remove them.