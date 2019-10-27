# Readme

In order to deploy vault in HA mode (3 pods by default), with consul as a backend, you only need to apply the provided resources in the vault directory.

For the aim of simplicity and a minimum respect of opesnhift security best practices, you need to create a specific serviceaccount that can run vault image as root. This is due to the vault image that runs with root as a user by default.

```bash
oc create serviceaccount vault-user
oc adm policy add-scc-to-user anyuid -z vault-user
```

Once the vault-user serviceaccount is created, apply the openshift resources with following commands:

```bash
oc apply -f vault-cm.yaml
oc apply -f vault-svc.yaml
oc apply -f vault-deploy.yaml
```

Finally create a route to the vault service:

```bash
oc create route passthrough vault \
    --service vault --port api \
    --hostname vault.secret-mgmt.$(minishift ip).nip.io
```

## Demo

Unseal all vault instances

```bash
for i in `oc get po | grep vault | awk '{ print $1}'`; do oc exec $i -- vault unseal -tls-skip-verify <Root Token>; done;
```

Add secrets to vault

```bash
export VAULT_ADDR=https://`oc get route -n secret-mgmt | grep -m1 vault | awk '{print $2}'`
vault login -tls-skip-verify <Root-Token>
vault kv put -tls-skip-verify secret/demo/app user=abdessamad passwd=pass
vault kv get -tls-skip-verify secret/demo/app
```

This will add your secrets to vault. The last command is to check them in secret/demo/app path.
Let's add now a new serviceaccount that'll be allowed to get secret from vault using the kubernetes auth mechanism.

```bash
oc create sa demo-vault -n demo-app
oc adm policy add-cluster-role-to-user system:auth-delegator -z demo-vault -n demo-app
vault auth enable -tls-skip-verify kubernetes
```

Now give a read access to this serviceaccount, by creating a custom policy.

```bash
export SA_TOKEN=$(oc get sa/demo-vault -o yaml -n demo-app | grep demo-vault-token | awk '{print $3}')
export SA_JWT_TOKEN=$(oc get secret $SA_TOKEN -n demo-app -o jsonpath="{.data.token}" | base64 --decode; echo)
export SA_CA_CRT=$(oc get secret $SA_TOKEN -n demo-app -o jsonpath="{.data['ca\.crt']}" | base64 --decode; echo)

vault write -tls-skip-verify auth/kubernetes/config \
  token_reviewer_jwt="$SA_JWT_TOKEN" \
  kubernetes_host="$(oc whoami --show-server)" \
  kubernetes_ca_cert="$SA_CA_CRT"

vault policy write -tls-skip-verify demo-policy-static demo-app-policy-static.hcl
vault write -tls-skip-verify auth/kubernetes/role/demo-app \
  bound_service_account_names=demo-vault \
  bound_service_account_namespaces=demo-app \
  policies=demo-policy-static \
  ttl=24h
```
