---
layout: post
title:  "Securing Azure Service Principals using Hashicorp Vault"
author: Prabhu
categories: [ Azure ]
image: assets/images/AHV.jpg
featured: true
---

Azure Service Principals are identities used to access Azure resources by non-Azure clients such as automation tools, applications, etc.  

Even though Azure provides options to restrict the access for a service principal to a particular resource or group or action, there is still a risk of being exposed. So we will see how to secure the service principal using Hashicorp vault by adding an additional layer of authentication and dynamic service principal generation.

Hashicorp Vault provides us options to dynamically generate service principals with a specific TTL. Applications / tools need to authenticate to Vault first which will retrieve a new principal. The same principal cannot be used once the TTL is over hence adding an additional security wrapper.


![Architecture](/assets/images/2020-08-30/img1.JPG)

For this demo, i'm going to install and configure the vault on a Ubuntu VM. And we need a service principal which will be used by the Vault to create dynamic SPs. This SP will need the below API permissions.

![Grant access](/assets/images/2020-08-30/img2.JPG)

Make sure the below API permissions are granted admin consent which can be done only by Organization's administrator. 

![API Permission](/assets/images/2020-08-30/img3.JPG)

Lets get started with the setup.

## Install Vault and Consul

Install the vault using below commands
```
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install vault
```

Install consul and start it
```
wget https://releases.hashicorp.com/consul/1.8.0/consul_1.8.0_linux_amd64.zip
apt install unzip
unzip consul_1.8.0_linux_amd64.zip
cp consul /usr/local/bin/
consul agent -dev > consul.log &
```

## Start Vault and unseal

Create a config.hcl file with the below configuration
```
storage "consul" {
  address = "127.0.0.1:8500"
  path    = "vault/"
}

listener "tcp" {
 address     = "127.0.0.1:8200"
 tls_disable = 1
}

```

Start Vault with the configuration
```
vault server -config=config.hcl > vault.log &
```

Initialize the operator.
```
export VAULT_ADDR='http://127.0.0.1:8200'
vault operator init
```

This will display the list of 5 keys and root token.

![Token](/assets/images/2020-08-30/img4.JPG)

## Configure Vault for Azure

Unseal the vault. Issue the below command 3 times and provide the unseal key. Use different key everytime.
```
vault operator unseal
```

Login to Vault. Provide the root token.
```
vault login
```

Enable Azure secret engine
```
vault secrets enable azure
```

Configure Azure service principal in Vault
```
export AZURE_SUBSCRIPTION_ID='**************************'
export AZURE_TENANT_ID='***************************'
export AZURE_CLIENT_ID='***************************'
export AZURE_CLIENT_SECRET='****************************'

vault write azure/config \
subscription_id=$AZURE_SUBSCRIPTION_ID \
tenant_id=$AZURE_TENANT_ID \
client_id=$AZURE_CLIENT_ID \
client_secret=$AZURE_CLIENT_SECRET
```

Create a role for a particular resource group with Contributor access
```
vault write azure/roles/v-contributor ttl=1h azure_roles=-<<EOF
    [
        {
            "role_name": "Contributor",
            "scope":  "/subscriptions/*********************/resourceGroups/rg-test"
        }
    ]
EOF
```

Create a policy to read the secrets
```
vi azure-policy.hcl
# Get credentials from the azure secrets engine
path "azure/creds/*" {
  capabilities = [ "read" ]
}

vault policy write azure-policy azure-policy.hcl
```

Create token for the policy
```
vault token create -policy=azure-policy
```

This will give us the token details.

![Azure token](/assets/images/2020-08-30/img5.JPG)

## Retrieve Dynamic Service Principal

Now that we have the token, we can use it to authenticate and retrieve the dynamic service principal.

```
VAULT_TOKEN=s.dV*************** vault read azure/creds/v-contributor

Key                Value
---                -----
lease_id           azure/creds/v-contributor/AeY3ckPy2pByrVtAasdfWADFasdf
lease_duration     1h
lease_renewable    true
client_id          ************************
client_secret      ************************
```

client_id and client_secret retrieved can now be used to authenticate to Azure. This will be active for 1 hour and cannot be reused after that. New dynamic SP has to be request again.

## Using API calls

Vault authentication and SP retrieval can also be automated by using API calls as below.

```
curl --header "X-Vault-Token: s.dV***************" \
       --request GET \
       http://127.0.0.1:8200/v1/azure/creds/v-contributor | jq
{
  "request_id": "787a8f6e-ccee-f330-de19-2c5ecd752878",
  "lease_id": "azure/creds/v-contributor/AeY3ckPy2pByrVtAasdfWADFasdf",
  "renewable": true,
  "lease_duration": 3600,
  "data": {
    "client_id": "**************************",
    "client_secret": "*************************"
  },
  ...
}
```
Using this API call any external application can authenticate to vault and retrieve SP which it can use to authenticate to Azure.

#### This is a simple setup of adding an additional layer of restrictions to the Azure service principals. The use-case can be extended even further by introducing IP restrictions for any Azure private connections !!! Lets keep exploring !!! Cheers !!!