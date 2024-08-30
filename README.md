# Control Center SSO for CFK with Azure AD

[Confluent Platform 7.7](https://docs.confluent.io/platform/current/release-notes/index.html) and [CFK 2.9.0](https://docs.confluent.io/operator/current/release-notes.html) released C3 SSO without any LDAP requirement.
This repository sets up a basic CP cluster via CFK running on Azure Kubernetes Service (AKS) enabling
users to log in via Azure AD (Entra ID) as an SSO.

In technical detail it deploys:
* 1 KraftController / 3 ZK nodes
* 3 Kafka brokers (including MDS)
* 1 KafkaRestClass
* 1 Control Center

> [!WARNING]
> There is a bug in CP 7.7: The JTI claim is required within the JWT token which should be originally optional.
> In Azure, you cannot easily update the claims.
> Therefore, we need to wait for CP 7.7.1.

General resources:
* [Quickstart: Deploy an Azure Kubernetes Service (AKS) cluster using Azure portal](https://learn.microsoft.com/en-us/azure/aks/learn/quick-kubernetes-deploy-portal?tabs=azure-cli)
* [Confluent for Kubernetes Quick Start](https://docs.confluent.io/operator/current/co-quickstart.html)
* [Single sign-on authentication for Confluent Control Center](https://docs.confluent.io/operator/current/co-authenticate-cp.html#single-sign-on-authentication-for-c3)

## Azure AD endpoints

For the later configuration we need to set the `token_endpoint`, the `jwks_uri`, and the `issuer`.
We can obtain all information via

```shell
curl https://login.microsoftonline.com/<tenant-id>/v2.0/.well-known/openid-configuration | jq
```

Generally, those are
```
token_endpoint = https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token
jwks_uri = https://login.microsoftonline.com/<tenant-id>/discovery/v2.0/keys
issuer = https://login.microsoftonline.com/<tenant-id>/v2.0
```

## Azure AD applications

To retrieve the JWT token, CP is using the client credentials grant flow. So, we need to register an application in Azure AD
and create a secret.
We can get a JWT token via:
```
curl -X POST -H "Content-Type: application/x-www-form-urlencoded" \
-d 'client_id=[client_id]&client_secret=[client_secret value]&grant_type=client_credentials' \
https://login.microsoftonline.com/[tenant_id]/oauth2/token
```

## CFK cluster

* [Configure RBAC for Confluent Platform Using Confluent for Kubernetes](https://docs.confluent.io/operator/current/co-rbac.html#requirements-and-considerations)

We need to configure CFK with RBAC to enable SSO.
This includes configuring MDS using OAuth.

```shell
Create OAuth secret 

kubectl create -n confluent secret generic oauth-jass \
 --from-file=oidcClientSecret.txt=oidcClientSecret.txt \
 --from-file=oauth.txt=oidcClientSecret.txt
```


Check out: https://docs.confluent.io/operator/current/co-rbac.html#migrate-rbac-from-using-ldap-to-using-both-ldap-and-oauth

### Configure Kafka CR

We configure MDS with OAuth and SSO.

```shell
Create MDS Token (evtl. not needed) !!

kubectl create secret generic mds-token \
--from-file=mdsPublicKey.pem=./MDS/mds-publickey.txt \
--from-file=mdsTokenKeyPair.pem=./MDS/mds-tokenkeypair.txt \
-n confluent
```


```yaml
Add to Kafka CR
```

### Configure KafkaRestClass CR

We need to set the authentication to OAuth.

```yaml
Add KafkaRestClass CR
```

### Configure C3 CR

We need to set the authentication to OAuth and to specify SSO

```yaml
Add ControlCenter CR
```

### TLS config

I am not sure if that is needed

```shell
Create certificates
https://github.com/confluentinc/confluent-kubernetes-examples/tree/master/assets/certs

openssl genrsa -out ./TLS/rootCAkey.pem 2048

openssl req -x509  -new -nodes \
  -key ./TLS/rootCAkey.pem \
  -days 3650 \
  -out ./TLS/cacerts.pem \
  -subj "/C=US/ST=CA/L=MVT/O=TestOrg/OU=Cloud/CN=TestCA"

cfssl gencert -ca=./TLS/cacerts.pem \
-ca-key=./TLS/rootCAkey.pem \
-config=./TLS/ca-config.json \
-profile=server ./TLS/server-domain.json | cfssljson -bare ./TLS/server


Create secret with certificates
kubectl create secret generic tls-group1 \
--from-file=fullchain.pem=./TLS/server.pem \
--from-file=cacerts.pem=./TLS/cacerts.pem \
--from-file=privkey.pem=./TLS/server-key.pem \
-n confluent
```



### Should not be needed because it is only used for Bearer

```
REST Credentials

kubectl create secret generic rest-credential \
--from-file=bearer.txt=./MDS/bearer.txt \
-n confluent
```

```
kubectl create secret generic mds-client \
--from-file=bearer.txt=./MDS/bearer.txt \
-n confluent
```

## Set Callback URL

Redirect callback URL to C3 added in the client app.
C3 Redirect URL format:
`https://<c3-host-name>:<c3-port-number>/api/metadata/security/1.0/oidc/authorization-code/callback`
