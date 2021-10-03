# Data Encryption Keys
The API Server stores data in etcd. You need to encrypt them. There's a number of encryption options, such as `identity`, `aescbc`, `kms` etc. You can read in detail from the [official document](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/). 

## The Encryption Key

In order to encrypt data stores in etcd, you need a `EncryptionConfiguration` Kubernetes object. It contains a base64 encoded secret. This step generates a random string and encode with base64. 

I won't explain too much about the command used in this step. Basically, you can think `/dev/urandom` as a random number generator.

```bash
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

## The Encryption Config File

The encryption config YAML file is used in [Chapter 08](https://github.com/prabhatsharma/kubernetes-the-hard-way-aws/blob/master/docs/08-bootstrapping-kubernetes-controllers.md#configure-the-kubernetes-api-server) as in flag `--encryption-provider-config` when start API Server. As mentioned previously, there's a number of encryption providers and here we use `aescbc`.
