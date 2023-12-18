---
title: "Vault for rotating database credentials"
date: 2023-12-18
tags: "vault,security,postgres,redis"
---

# Vault for rotating database credentials

Vault is a powerful tool for managing secrets. It can be used to manage secrets for applications, infrastructure, and even databases. Here I will share some notes from my exploration about how we can use vault for managing database credentials used by applications.

Key important requirements to me when it comes to managing secret for application is:
* Secret should be rotated periodically
* Application should be able to immediately use the new secret after rotation

In this case, I will take redis as example. The same concept can be applied to other databases as well (e.g. postgresql).

We have this `docker-compose.yml` which will create a new redis container:

```yaml
version: '3.9'
services:
  redis:
    image: bitnami/redis:7.0
    environment:
      - ALLOW_EMPTY_PASSWORD=no
      - REDIS_PASSWORD=adminpassword
    volumes:
      - redis_data:/bitnami
    ports:
      - "6379:6379"
volumes:
  redis_data:
``````

The docker compose above will create `default` redis user with password `adminpassword`. However, redis has its own ACL system which can be used to manage access to redis. It is possible to create user with limited access to specific database and do more granular acl for users. For more, you can read [here](https://redis.io/docs/management/security/acl/). 

## Vault Database Engine

You can refer to the official documentation on how to do this by using vault cli [here](https://developer.hashicorp.com/vault/docs/secrets/databases/redis). However, I will show you how to do this using terraform.

To start, we should start with creating a new mount for redis database engine. This secret mount can be used to store all redis instances which will be managed by vault. For instance, I want to group all redis instances deployed on my gcp project named `imrenagicom`. I can create a new mount named `redis-imrenagicom`. 

```hcl
resource "vault_mount" "redis_secret_mount" {
  path = "redis-imrenagicom" 
  type = "database"
}
```

Then I should create a new connection to redis instance. This connection will be used by vault to connect to redis instance. However, before we can do this, we should create a root user for vault. This is not the user which will be used by application. This is just a user which will be used by vault to manage user, rotate credentials, etc. We actually can use `default` user with password `adminpassword` created by redis docker container earlier, but assuming that we want to create new dedicated user for vault named `vault` and initial password `vault`.

```bash
redis-cli ACL SETUSER vault on >vault ~* &* +@all
```

Then to create redis connection, we use this:

```hcl
resource "vault_database_secret_backend_connection" "redis_testing" {
  backend       = vault_mount.redis_secret_mount.path
  name          = "ticketing-service-cache"
  plugin_name   = "redis-database-plugin"
  allowed_roles = [
    "ticketing-app-role",
  ]
  redis {
    host      = "10.1.20.123"
    port      = 6379
    username  = "vault"
    password  = "vault"
    tls       = false
  }
}
```

The resource above creates a new vault backend connection named `ticketing-service-cache` with some credential information for redis. The `allowed_roles` is used to define which roles can be used by this connection. In this case, I have two roles named `admin-role` and `ticketing-app-role`. This `allowed_roles` is a bit confusing to me at first. Turned out, this is the vault role that can use within this backend connection. More on this later.

Once we have this, we can choose whether we want to rotate the root credentials. When the credential is rotated, initial `vault` password can't be used anymore and only vault which know the new password. To do this, we can use this command:

```bash
$ vault write redis-imrenagicom/rotate-root/ticketing-service-cache
```

In vault, we can either create dynamic secret or static secret. Dynamic secret can be used to create new credential for each request to vault. On the other hand, static secret can be used to managed existing redis user. Both secret can be configured with limited ttl and rotation period. 

Let's start with static secret first. We will create another redis user which will be used by the application to connect to redis. This user name is `jokowi` with initial password `gibrananakkusayang`. This user will set all access to all keys and pub/sub channel.

```bash
redis-cli ACL SETUSER jokowi on >gibrananakkusayang ~* &* +@all
```

Once we have this, we will define a new backend static role to manage this user. In addition, we also configure the rotation period to `30d`. This means that vault will rotate the password for this user every 30 days. Please note that after rotation, user scope will stay the same. This means that the user will still have access to all keys and pub/sub channel.

```hcl
resource "vault_database_secret_backend_static_role" "period_role" {
  backend             = vault_mount.redis_secret_mount.path
  name                = "ticketing-app-role"
  db_name             = vault_database_secret_backend_connection.redis_testing.name
  username            = "jokowi"
  rotation_period     = "30d"
}
```

Once this role is applied, we can read the credential for this user by using this command:
```bash
$ vault read redis-imrenagicom/static-creds/ticketing-app-role

Key                    Value
---                    -----
last_vault_rotation    2023-12-11T10:39:49.647822-06:00
password               ylKNgqa3NPVAioBf-0S5
rotation_period        30d
ttl                    4m39s
username               jokowi
```

Okay now we have this credential automatically rotated. Now what we can do to ensure this credential gets updated to the application when it is rotated?

## Synchronizing Vault secret to application

This really depends on how you deploy your application and how your application use the secret. 

In general, if you are running your application on a VM, you can use Vault SDK to retrieve the credentials or use Vault agent to write the credential to a file. You can use Vault SDK to load all required secret during application startup or load it lazily when the secret is needed. You can also use Vault agent to write the credential to a file and then read it from your application. Vault agent has capability to update the secret stored in a file when it detects secret change/rotation. However, please remember that if you only read the secret during the startup time, you still need to restart the application to ensure that the new credential is used.

This approach makes sense if you have longer rotation period because you don't need to restart the application frequently. If the rotation period is shorter, you might want to setup scheduler to restart the application once the secret is rotated and this seems like adding unnecessarily complexity at least to me.

When it comes to kubernetes, at least there are two possible approach that we can choose from. First is by using [Vault agent injector](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-sidecar) and the second is by using [Vault Secret Operator](https://www.hashicorp.com/blog/vault-secrets-operator-a-new-method-for-kubernetes-integration) (available only after Vault 1.13.x). The nice thing about these approach is that the application becomes unaware of Vault SDK since both approach will inject the secret to the application either as environment variable or as a file. 

The difference between these two approach is that Vault agent injector will create a new init and sidecar container which will be used to retrieve the secret from vault and then inject it to the application container. On the other hand, Vault Secret Operator will run custom resource controller watching CRD which configures how secret from vault can be synchronized to native kubernetes secret.

### Vault Agent Injector

Pros:
* Simple configuration by adding annotation to Pod or Deployment.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: payroll
    labels:
      app: payroll
    annotations:
      vault.hashicorp.com/agent-inject: "true"
      vault.hashicorp.com/role: "reader-role"
      vault.hashicorp.com/agent-inject-secret-app-creds: "secret/random-app"
      vault.hashicorp.com/agent-inject-template-app-creds: |
        {{- with secret "secret/random-app" -}}
        db_password: {{ .Data.data.db_password }}
        db_user: {{ .Data.data.db_user }}
        {{- end -}}
  spec:
    serviceAccountName: app
    containers:
      - name: payroll
        image: stefanprodan/podinfo
  ```

Cons:
* It adds a sidecar container pods. Hence, it requires more resources to run.
* While vault agent sidecars allow secrets stored on the file to get updated, no mechanism available to restart the deployment to update the secret. Thus, secret load can only be done during runtime by re-reading the files or watching the files for update.
* Requires effort to manage kubernetes agent injector service and its mutating webhook 

### Vault Secret Operator

Pros:
* No sidecar container attached to the pod thus reducing resource consumption.
* Automatic synchronization and use native kubernetes secret for storing the secret from the vault. Thus, applications can use the vault secret either as mounted volume or environment variable.
* The Operator can also perform post-rotation actions, like notifying an application directly by triggering a rolling update of a deployment. This allows the application to refresh the secret it loads during startup time.
* Support for dynamic secret is available

Cons:
* It is still beta and require at least version 1.13.x
* Requires effort to manage additional kubernetes CRD for each application
