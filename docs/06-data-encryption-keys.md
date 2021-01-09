# Generating the Data Encryption Config and Key for kube-apiserver to encrypt data in etcd (i.e. provide encryption at rest by encrypting etcd data)

Kubernetes stores a variety of data including cluster state, application configurations, and **secrets** in etcd in plaintext. Kubernetes supports the ability to [encrypt](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data) cluster data at rest (i.e. encrypt data in etcd by configuring the required parameter in kube-apiserver).

```
Ref: https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#configuration-and-determining-whether-encryption-at-rest-is-already-enabled

The kube-apiserver process accepts an argument --encryption-provider-config that controls how API data is encrypted in etcd
```

In this lab you will generate an encryption key and an [encryption config](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#understanding-the-encryption-at-rest-configuration) suitable for encrypting Kubernetes Secrets.

## The Encryption Key

Generate an encryption key:

```
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

## The Encryption Config File

Create the `encryption-config.yaml` encryption config file:

```
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

> In this case, resources are ONLY set to **`secrets` (i.e. only secrets object in etcd are encrypted)**

Copy the `encryption-config.yaml` encryption config file to each controller instance:

```
for instance in master-1 master-2; do
  scp encryption-config.yaml ${instance}:~/
done
```
Reference: https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#encrypting-your-data

Next: [Bootstrapping the etcd Cluster](07-bootstrapping-etcd.md)
