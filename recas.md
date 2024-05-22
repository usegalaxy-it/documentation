# Access to ReCaS Cloud with token

In the following we introduce two token:

1. IAM access Token: it is a token issued by IAM, the AAI system used by ReCaS Cloud.

2. Keystone Token: it is a token issued by Keystone, the identity manager of OpenStack, which can be configured to accept IAM token. Indeed, it is possible from a IAM token to restrieve a Keystone token.

## Pre-requisites

In order to check if the authentication on ReCaS Cloud is working we will use openstack client. So:

1. Install a python virtual environment, which is called "venv", and activate it:
```
python3 -m venv venv
. ./venv/bin/activate
```

2. Install the openstack client
```
pip install openstackclient
```

## Access with IAM Token

IAM is the Identity provider (IdP) used by ReCaS Cloud. We will use oidc-agent to interact with IAM.

### Install oidc-agent

The installation and configuration of oidc-agent is well explained [here](https://github.com/indigo-dc/oidc-agent).

We recommend to use the "device" flow for creating the a new IAM client with oidc-agent.

In the following the short name of the client created with oidc-gen will be ``iam-recas``.

### Get IAM token and access to cloud resources

1. Run oidc-agent:

   ```
   eval `oidc-agent-service use`
   oidc-add iam-recas
   ```
2. Create OpenStack tenant rc file with your favourite editor, ``vim yourtenant.sh``, with the following parameters:

   ```
   #!/usr/bin/env bash
   export OS_AUTH_URL=https://keystone.recas.ba.infn.it/v3
   export OS_AUTH_TYPE=v3oidcaccesstoken
   export OS_PROJECT_ID=<openstack tenant id>
   export OS_TENANT_ID=<openstack tenant id>
   export OS_PROTOCOL="openid"
   export OS_IDENTITY_PROVIDER="recas-bari"
   export OS_IDENTITY_API_VERSION=3
   export OS_REGION_NAME="RegionOne"
   export OS_INTERFACE=public
   export OIDC_AGENT_ACCOUNT=iam-recas
   export OS_ACCESS_TOKEN=$(oidc-token ${OIDC_AGENT_ACCOUNT})
   export OS_AUTH_TOKEN=$OS_ACCESS_TOKEN
   ```

3. Source the rc file:

   ```
   . yourtenant.sh
   ```

4. Test the access:

   ```
   openstack image list
   ```

    and you should be able to see openstack images available on your tenant.

## Get a Keystone Access token 

It is possoble, and needed for example to use Hashicorp Terraform for cloud deployment, to retrieve a Keystone token.

1. Run oidc-agent and get a IAM token:

   ```
   eval `oidc-agent-service use`
   oidc-add iam-recas
   oidc-token iam-recas
   ```

   Then assign the token you retrieved from oidc-token command to a variable:

   ```
   export IAM_ACCESS_TOKEN=<token from oidc-token command>
   ```

2. Create OpenStack tenant rc file with your favourite editor, ``vim yourtenant.sh``, with the following parameters:

   ```
   #!/usr/bin/env bash
   export OS_AUTH_URL=https://keystone.recas.ba.infn.it/v3
   export OS_AUTH_TYPE=token
   export OS_PROJECT_ID=<openstack tenant id>
   export OS_TENANT_ID=<openstack tenant id>
   export OS_PROTOCOL="openid"
   export OS_IDENTITY_PROVIDER="recas-bari"
   export OS_IDENTITY_API_VERSION=3
   export OS_REGION_NAME="RegionOne"
   export OS_INTERFACE=public
   ```

3. Source the rc file:

   ```
   . yourtenant.sh
   ```

4. Get a keystone token

   ```
   openstack --os-auth-type v3oidcaccesstoken --os-access-token ${IAM_ACCESS_TOKEN} --os-auth-url https://keystone.recas.ba.infn.it/v3 --os-protocol openid --os-identity-provider recas-bari --os-identity-api-version 3 --os-project-id <your tenant id> token issue
   ```

5. Assign the keystone token, which is the ``id`` field from the output of the previous command to the environment variable ``OS_TOKEN``:

   ```
   export OS_TOKEN=<keystone token from id field>
   ``` 
6. Test the access:

   ```
   openstack image list
   ```

    and you should be able to see openstack images available on your tenant.

## Terraform hints

To use Hashicorp Terraform with ReCaS cloud, it is needed to follow the section [for retrieving KeyStone access token](#get-a-keystone-access-token). Moreover you need to assign the KeyStone token to the variable ``OS_AUTH_TOKEN`` which is the one used by terraform:

   ```
   export OS_AUTH_TOKEN=<keystone token from id field>
   ```
Finally, due to the configuration of ReCaS Cloud endpoints and a bug in the terraform library to interact with openstack, it is needed to add to the providers.tf file the following endpoint map:

```
# Configure the OpenStack Provider
provider "openstack" {
  tenant_id = "<openstack tenant id>"
  auth_url = "https://keystone.recas.ba.infn.it/v3"
  endpoint_overrides = {
    "network"  = "https://neutron.recas.ba.infn.it/v2.0/"
    "volumev3" = "https://cinder.recas.ba.infn.it/v3/"
    "image" = "https://glance.recas.ba.infn.it/v2/"
  }
}
```
