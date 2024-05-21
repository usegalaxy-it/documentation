# Access to ReCaS Cloud with token

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

IAM is the Identity provider (IdP) used by ReCaS Cloud. We will use oidc-agent to interact with IAM

The installation and configuration of oidc-agent is well explained [here](https://github.com/indigo-dc/oidc-agent).

We recommend to use the "device" flow for creating the a new IAM client with oidc-agent.

In the following the short name of the client created with oidc-gen will be ``iam-recas``

## Get a Keystone Access token 

## Terraform hints
