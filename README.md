# BPDM Association Environments

This repository contains configuration files and information to our current running environments of the BPDM Chart found in the [eclipse-tractusx/bpdm](https://github.com/eclipse-tractusx/bpdm) repository.

## Int

Available at: https://argocd.int.catena-x.net

| App                       | Chart Version               | Root URL                                                                 |
|---------------------------|-----------------------------|--------------------------------------------------------------------------|
| bpdm                      | 5.3.0                       | https://business-partners.int.catena-x.net/                              | 
| bpdm-test-sharing-member  | 5.3.0                       | https://business-partners.int.catena-x.net/companies/test-sharing-member | 
| bpdm-sap                  | 5.3.0                       | https://business-partners.int.catena-x.net/companies/sap                 | 
| bpdm-snapshot             | main                        | https://business-partners-snapshot.int.catena-x.net/                     |
| bpdm-hq-relocation        | feat/headquarter-relocation | https://business-partners-hq-relocation.int.catena-x.net/                |
| bpdm-provider-edc         | 0.11.1                      | https://bpdm-edc.int.catena-x.net                                        | 
| test-sharing-member-edc   | 0.11.1                      | https://bpdm-sharing-member-edc.int.catena-x.net                         | 
| test-sharing-member-2-edc | 0.11.1                      | https://bpdm-sharing-member-edc-2.int.catena-x.net                       | 

## Stable

Available at: https://argocd.stable.catena-x.net

| App                      | Chart Version | Root URL                                                                    |
|--------------------------|---------------|-----------------------------------------------------------------------------|
| bpdm                     | 5.2.0         | https://business-partners.stable.catena-x.net/                              | 
| bpdm-test-sharing-member | 5.2.0         | https://business-partners.stable.catena-x.net/companies/test-sharing-member | 
| bpdm-sap                 | 5.2.0         | https://business-partners.stable.catena-x.net/companies/sap                 |
| bpdm-provider-edc        | 0.8.0         | https://bpdm-edc.stable.catena-x.net                                        | 
| test-sharing-member-edc  | 0.8.0         | https://bpdm-sharing-member-edc.stable.catena-x.net                         | 

# Environment Setup Walkthrough

In case you want to fully deploy BPDM in the Catena-X association environment you can follow these steps.

Deployments on the association environment are being done by [Helm Charts](https://helm.sh/) that are configured and deployed on [ArgoCD](https://argo-cd.readthedocs.io/).

At the moment the association offers two environments: [INT](https://argocd.int.catena-x.net/) and [STABLE](https://argocd.stable.catena-x.net/).

In the following sections let us assume we want to set up BPDM for the STABLE environment.

## Set up the Golden Record Process

First we want to set up the golden record core process which is needed by the [Portal](https://portal.stable.catena-x.net) to onboard companies and offer business partner services. 

### Create Secrets

In order to set up the golden record process you first need to create the necessary secrets for database access and the access between the single BPDM components.

For configuring the secrets of our STABLE environment we need to add database and client-credential secret values.
The database secret values are free to choose.
However, the client credential secrets are dependent on the [STABLE Central-IDP](https://centralidp.stable.catena-x.net/auth/) configuration.
For the STABLE environment BPDM relies on deployed Central-IDP and Portal applications from which we can gather the necessary client credentials.

In order to do so, you need to have an account in the STABLE Portal for the `Cx-Operator` company.
Reach out to the team responsible for managing the association's STABLE Portal to get an invitation.

Once you have the account go to the technical user management page and gather the credentials for the following already existing clients:

- sa-cl7-cx-1: Portal Gate's client to access the Pool
- sa-cl25-cx-1: Cleaning Service Dummy's client to access the Orchestrator
- sa-cl25-cx-2: Pool's client to access the Orchestrator
- sa-cl25-cx-3: Gate's client to access the Orchestrator

For the managing the secrets you want to have a secure way to store and use them in your Helm Chart deployments.

For that reason association offers a [hashicorp vault](https://www.hashicorp.com/de/products/vault) for secret management.
The [association's vault](https://vault.core.catena-x.net) is a central service independent of any single environment and should hold secrets for every environment.

The association's argo-cd comes with a plugin that is able to access the vault and use secrets referenced in the Helm Chart values.yaml file.
For argo-cd to identify these secret references the values need to be specified in the given pattern:

\<path:bpdm/data/environment/path/to/secret#secret-value>

- `bpdm` is the secret engine in which all BPDM secrets are stored.
  The secret engine is created by the association's administrators.
- `data` is an API path for accessing secrets of an engine (as opposed to `configuration` for example).
  Note that it will not be shown in the vault's UI and should not be added to the path when creating secrets.
- `environment` is the name of the environment for which the secret is for.
- `/path/to/secret` is the actual path of the secret.
  It should be the same for each environment.
- `#secret-value` is the accessor to the actual secret value.
  Each secret can hold several named values.

You can look in the repository's [values.yaml](./stable/bpdm/values.yaml) for examples.

Next, we want to store our secret values under the following paths in the vault:

- stable/bpdm/postgresql#postgres-password (free to choose)
- stable/bpdm/postgresql#password (free to choose)
- stable/bpdm/gate/pool-client#client-id (from sa-cl7-cx-1)
- stable/bpdm/gate/pool-client#client-secret (from sa-cl7-cx-1)
- stable/bpdm/gate/orchestrator-client#client-id (from sa-cl25-cx-3)
- stable/bpdm/gate/orchestrator-client#client-secret (from sa-cl25-cx-3)
- stable/bpdm/pool/orchestrator-client#client-id (from sa-cl25-cx-2)
- stable/bpdm/pool/orchestrator-client#client-secret (from sa-cl25-cx-2)
- stable/bpdm/cleaning-service/orchestrator-client#client-id (from sa-cl25-cx-1)
- stable/bpdm/cleaning-service/orchestrator-client#client-secret (from sa-cl25-cx-1)

Each secret path's are organized in the following way:

- `stable` is the root path for all our secrets on the STABLE environment
- `bpdm`: the name of the deployment for which these secrets are
- `postgresql` the secrets for the database. Our golden record components in one deployment share one database
- `gate`: secrets used for the BPDM gate component
- `pool`: secrets used for the BPDM pool component
- `cleaning-service`: secrets used for the BPDM cleaning service dummy component

These are the secrets needed to deploy a working golden record process for the [Portal](https://portal.stable.catena-x.net).
For deploying and configuring an EDC and additional gates for sharing members we will need more secrets.
More to that in the respective sections.

### Deploy the BPDM Chart

After we have created the necessary secrets for the core golden record process it is time to deploy the BPDM applications.
As mentioned before we will use the BPDM Helm Chart and argo-cd to deploy.

1. Go to the association's STABLE [argo-cd](https://argocd.stable.catena-x.net/)
2. Create a `New App`
3. Give the app a name and use the argo-cd [deployment spec](./stable/bpdm/spec.yaml)
4. Create the app

This will create a new argo-cd app which deploys the BPDM Chart configured with the [STABLE values.yaml](./stable/bpdm/values.yaml).

After that step argo-cd lets you preview the Kubernetes components which will be deployed (indicated by the yellow state).
Feel free to verify the components.
Once you are sure everything looks right, you can click on the `Synchronize` button to finally deploy the BPDM core components.
It may take while but after a few minutes the app should go into a green state.

If everything was deployed correctly, the Portal should now be able to onboard new companies.
Also, the Portal's partner network and company data page should be working.

### Operator API Access

As an operator of the STABLE BPDM deployment you may want to access the BPDM API's directly for debugging reasons.
You can create yourself technical users over the Portal as a member of the Cx-Operator company.

For that, 
1. go to the Portal's technical user management page
2. Create a new technical user
3. Give it a fitting name and description for its purpose
4. Mark the BPDM permissions to get access to the BPDM APIs

For the core golden record process there are three APIs. 
You can get full access with the following permissions:

1. Pool: `BPDM Pool Admin`
2. Portal Gate: `BPDM Sharing Admin`
3. Orchestrator: `BPDM Orchestrator Admin`

## Set up a sharing member Gate

The golden record process is now available for the Portal and its users.
However, sharing members need their own separate Gates.
For our use case we will onboard a new test sharing member and deploy a Gate for them.

### Create Client Credentials

When deploying the golden record core services the Portal already came with preconfigured client credentials we could use for our secrets.
This time however, we should create new client credentials for the Gate we want to deploy.
We will need two technical users with the following permissions:

1. Test Sharing Member Gate Pool Access: `BPDM Pool Sharing Consumer`
2. Test Sharing Member Gate Orchestrator Access: `BPDM Orchestrator Task Creator`

### Create Secrets

Just as before, we first need to create the necessary secrets in the vault.
The Gate deploys its own database so we need some free to choose database passwords.
We also need to add the client credentials form the previous step as secret values.

- stable/bpdm-test-sharing-member/postgresql#postgres-password (free to choose)
- stable/bpdm-test-sharing-member/postgresql#password (free to choose)
- stable/bpdm-test-sharing-member/gate/pool-client#client-id (from pool access client)
- stable/bpdm-test-sharing-member/gate/pool-client#client-secret (from pool access client)
- stable/bpdm-test-sharing-member/gate/orchestrator-client#client-id (from orchestrator access client)
- stable/bpdm-test-sharing-member/gate/orchestrator-client#client-secret (from orchestrator access client)

### Deploy the Test Sharing Member Gate

After the secrets are configured it is time to deploy the Gate via argo-cd.

Create a new app in argo-cd with an appropriate name and use the [test-sharing-member-spec](./stable/bpdm-test-sharing-member/spec.yaml) as deployment specification.
This deploys a BPDM Helm Chart configured by the [test sharing member values](./stable/bpdm-test-sharing-member/values.yaml) with only the Gate component and database activated.
After a short time app should go into a green state.

## Set up the Golden Record Process EDC

The test sharing member Gate is now deployed and ready to use.
However, sharing members will need to access their Gate over EDC communication.
Therefore, we need to configure and deploy a Catena-X operator EDC which offers the necessary [BPDM assets](https://github.com/eclipse-tractusx/bpdm/blob/main/docs/architecture/08_Crosscutting_Concepts.md#edc-communication).

### Create EDC asset clients

The deployed sharing member Gate is branded to the test sharing member's BPNL. 
In order for the EDC to access it, it needs technical users which have the identity of the test sharing member company.
The Portal provides a way to create such technical users by app subscriptions.
For that reason the BPDM operator first needs to create the necessary Portal apps in the marketplace for the test company to subscribe to.
The BPDM repository describes [how to do that](https://github.com/eclipse-tractusx/bpdm/blob/main/INSTALL.md#create-bpdm-marketplace-apps).

After the apps are registered at the marketplace the test sharing member company can subscribe to them.
After the subscription has been activated, the Portal will list the created clients with the technical users.

That way we can obtain from the Portal three client credentials:

1. BPDM Pool Consumer Client
2. BPDM Sharing Input Manager Client
3. BPDM Sharing Output Consumer Client

### Create EDC wallet client

the EDC also needs access to its company's wallet.
The Portal supports the creation of clients to access the wallet.
As a Cx-Operator account we can therefore go to the technical user page of the Portal and create new `external` user with the  permission `Identity Wallet Management`.
Typically, it takes a few moments before the credentials are created.

### Create Vault Secrets

The EDC needs an RSA private and public key for communication and expects these keys to be stored in the association vault.
You can freely create such a private/public key pair.

With all client credentials and keys in place we create the following secrets:

- stable/edc-bpdm/api#management-key (free to choose)
- stable/edc-bpdm/api#proxy-key (free to choose)
- stable/edc-bpdm/postgres#password (free to choose)
- stable/edc-bpdm/vault#token  (your vault token)
- stable/edc-bpdm/token-private-key#content (private RSA key)
- stable/edc-bpdm/token-public-key#content (public RSA key)
- stable/edc-bpdm/client-secret#content (from wallet client)
- stable/edc-bpdm/asset-secrets/test-sharing-member/pool-member-read#content (from Pool Consumer Client)
- stable/edc-bpdm/asset-secrets/test-sharing-member/input-full-access#content(from Sharing Input Manager Client)
- stable/edc-bpdm/asset-secrets/test-sharing-member/output-read-access#content(from Sharing Output Consumer Client)

Also note that all secrets having a `content` secret value are secrets that the EDC accesses during runtime from the vault directly.
They are not used for the argo-cd plugin.

### Deploy BPDM EDC

After all the secrets are created in the vault it is time to deploy the EDC.
As before, go to the STABLE argo-cd and create a new app.
Use the [edc deployment spec](./stable/edc-bpdm/spec.yaml).
This will deploy the BPDM provider EDC using the [provider-values.yaml](stable/edc-bpdm/values.yaml).

Note that the provider values might need to be adapted to include the correct wallet client-id.

### Creating EDC Assets

A final step for the setting up the EDC is the creation of assets over which the test sharing member can access the BPDM services.

The BPDM repository contains [instructions](https://github.com/eclipse-tractusx/bpdm/blob/main/INSTALL.md#creating-offers) and a [Postman collection](https://github.com/eclipse-tractusx/bpdm/blob/main/docs/postman/EDC%20Provider%20Setup.postman_collection.json) to help with this setup.
We just need to make sure to set the necessary collection variables:

1. PROVIDER_EDC_API_KEY: stable/bpdm/edc/api#management-key
2. PROVIDER_EDC_MANAGEMENT_API: https://bpdm-edc.stable.catena-x.net/management
3. GATE_API: https://business-partners.stable.catena-x.net/companies/test-sharing-member
4. POOL_API: https://business-partners.stable.catena-x.net/pool
5. TOKEN_URL: http://centralidp.stable.catena-x.net/auth/realms/CX-Central/protocol/openid-connect/token
6. CONSUMER_BPNL: The BPNL of the test sharing member
7. CLIENT_ID_POOL_CX_MEMBER_READ: from Pool Consumer Client
8. CLIENT_SECRET_PATH_POOL_CX_MEMBER_READ_CLIENT: stable/edc-bpdm/asset-secrets/test-sharing-member/pool-member-read
9. CLIENT_ID_GATE_SHARING_MEMBER_FULL_INPUT: from Sharing Input Manager Client
10. CLIENT_SECRET_PATH_GATE_SHARING_MEMBER_FULL_INPUT: stable/edc-bpdm/asset-secrets/test-sharing-member/input-full-access
11. CLIENT_ID_GATE_SHARING_MEMBER_READ_OUTPUT: from Sharing Output Consumer Client
12. CLIENT_SECRET_PATH_GATE_SHARING_MEMBER_READ_OUTPUT: stable/edc-bpdm/asset-secrets/test-sharing-member/output-read-access

Note that the variables for the client secret path need to contain the path in the vault where the secret can be found.

## Set up the Sharing Member EDC

If you want to test the connection over EDC for the test sharing member company you need to set up an additional EDC acting as the consumer.
This EDC simulates the test sharing member side and will have the identity of the test sharing member.

The steps to deploy such an EDC are analogous to the BPDM provider EDC set up.
However, test sharing member EDC just acts as a consumer and does not need assets.
This simplifies the process a bit since no asset clients and credentials are needed.
This results in the following steps:

1. Create a wallet client in the test sharing member company
2. Create vault secrets (minus the asset secrets)
3. Deploy the test sharing member EDC with its [deployment spec](stable/edc-test-sharing-member/spec.yaml) which uses the [test sharing member values](stable/edc-test-sharing-member/values.yaml)

The values yaml might need to be adapted to the test sharing member's BPNL and wallet client-id.

Note that the secrets in the vault should be under the follow paths:

- stable/edc-test-sharing-member/api#management-key
- stable/edc-test-sharing-member/api#proxy-key
- stable/edc-test-sharing-member/postgres#password
- stable/edc-test-sharing-member/vault#token
- stable/edc-test-sharing-member/token-private-key#content
- stable/edc-test-sharing-member/token-public-key#content
- stable/edc-test-sharing-member/client-secret#content

Also, for trying out the connection between BPDM provider EDC and test sharing member EDC use the BPDM repository's [consumer Postman Collection](https://github.com/eclipse-tractusx/bpdm/blob/main/docs/postman/EDC%20BPDM%20Consumer.postman_collection.json).
The collection variables should be initialized as follows:

1. CONSUMER_EDC_API_KEY: (from secret value stable/sharing-members/test/edc/api#management-key)
2. CONSUMER_EDC_MANAGEMENT_API: https://bpdm-sharing-member-edc.stable.catena-x.net/management
3. PROVIDER_EDC_DATASPACE_API: https://bpdm-edc.stable.catena-x.net/api/v1/dsp
4. PROVIDER_BPNL: BPNL00000003CRHK
5. CONSUMER_BPNL: BPNL of the test company
6. PROVIDER_EDC_PUBLIC_API: https://bpdm-edc.stable.catena-x.net/api/public

