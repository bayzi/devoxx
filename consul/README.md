# Readme

In order to deploy a consul cluster in openshift (kubenetes), you can execute following commands.

## Overview

- We'll be deploying a 3 node consul cluster using Statefulset
- Communication between Consul members is secured using TLS and encryption keys

First we need to generate TLS certificates using [cfssl](https://pkg.cfssl.org/) and [cfssljson](https://pkg.cfssl.org/). For this you either install them locally, or you can use the golang docker image. I'll use the second option.

Make sure to change directory to **consul/ca**, and adapt the consul/ca/consul-csr.json file for your needs.

```bash

docker run -it --entrypoint /bin/sh -v $(PWD):/go golang:1.13.1-alpine3.10

apk add git build-base

go get -u github.com/cloudflare/cfssl/cmd/cfssl

go get -u github.com/cloudflare/cfssl/cmd/cfssljson

cd /app

cfssl gencert -initca consul-csr.json | cfssljson -bare ca

cfssl gencert -ca=ca.pem \
        -ca-key=ca-key.pem \
        -config=ca-config.json \
        -profile=default consul-csr.json \
        | cfssljson -bare consul

```

Once these commands are executed you'll notice the creation of some files in the consul/ca directory. We'll use those files to create a secret in openshift.

I'd like to have a dedicated project (namespace) in openshift for vault and consul called secret-mgmt:

```bash
oc new-project secret-mgmt
```

We'll need to install consul client on our machine:

```bash
brew install consul
```

Create then this variable, wich'll be used in the consul secret:

```bash
GOSSIP_ENCRYPTION_KEY=$(consul keygen)

oc create secret generic consul \
  --from-literal="gossip-encryption-key=${GOSSIP_ENCRYPTION_KEY}" \
  --from-file=ca.pem \
  --from-file=consul.pem \
  --from-file=consul-key.pem
```

For the aim of simplicity and a minimum respect of opesnhift security best practices, you need to create a specific serviceaccount that can run vault image as root. This is due to the vault image that runs with root as a user by default.

```bash
oc create serviceaccount vault-user
oc adm policy add-scc-to-user anyuid -z vault-user

oc apply -f ../service.yaml
oc apply -f ../statefulset.yaml

```

Check the the state of consul pods until you get a running status:

```bash
oc get po -w
```

```
NAME                     READY   STATUS    RESTARTS   AGE
consul-0                 2/2     Running   0          38m
consul-1                 2/2     Running   0          38m
consul-2                 2/2     Running   0          38m
```
