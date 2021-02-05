# How is it working ?

This step use Hashicorp Vault:
* Dynamic secrets with database
* Encryption as a Service
* `Vault Agent` for Auto-Auth method (with AppRole) and renew the Vault token
* `envconsul` to populate secrets into environment variables

Each time you will go to the website, the application will use a couple of user/password for database access with a short TTL. This secret will be renew before the expiration of the TTL by `envconsul`.
If the secret is revoked, `envconsul` will request a new one to Vault.
If the token expired, `Vault Agent` will request a new one to Vault (re-auth).

It use also Encryption as a Service. Each time you will go to the website, the application will Encrypt value and store the encrypted data into the database.

## Initialisation

First, init your terraform folder:

```bash
$ docker run --rm -v $(pwd)/terraform:/app/ -w /app/ hashicorp/terraform:light init
```

Or if you can use Makefile: `make init`

## Infrastructure

As an Ops, you need to deploy the infrastructure:

```bash
$ docker-compose up
```

Or if you can use Makefile: `make infra`

### Service access

* Vault address: [http://127.0.0.1:8200](http://127.0.0.1:8200)

## Application

As an Dev, you need to deploy the infrastructure. In this case, using Vault, your application use [Approle](https://www.vaultproject.io/docs/auth/approle.html) and need **Role_ID** and **Secret_ID**.

Here how to retrieve **Role_ID** and **Secret_ID**:

```bash
$ role_id=$(docker run --rm -v $(pwd)/terraform:/app/ -w /app/ hashicorp/terraform:light output -raw approle_role_id)
$ secret_id=$(docker run --rm -v $(pwd)/terraform:/app/ -w /app/ hashicorp/terraform:light output -raw approle_secret_id)
```

And launch your application:

```
$ docker-compose -f app.yml run -e VLT_ROLE_ID=$role_id -e VLT_SECRET_ID=$secret_id --service-ports web
```

Or if you can use Makefile: `make app`

### Service access

* Application address: [http://127.0.0.1:8080](http://127.0.0.1:8080)

## Test

### Secret rotation

1. Connect with your web browser to the Vault URL
2. Use token connection and enter as a token: `root`
3. Go to `Access` path and select: `lease`
4. Go to `database` and the first lease you find
5. Revoke the lease ID.

### EaaS

Going into the website, you will find an encrypted data from Vault. We will decrypt this value to test if the EaaS working.
Your application can only encrypt and can not decrypt (check the [web.hcl](./terraform/web.hcl)).

Vault informations access for web UI:

1. Connect with your web browser to the Vault URL
2. Use token connection and enter as a token: `root`
3. Go to `transit` path and select: `web`
4. In `key actions`, select: `Decrypt`
5. Put your encrypted value and decrypt it
6. `Decode from base64` and you will get the decrypt value who should be equel to the server name

## Cleanup

Do the following commands:

```bash
$ docker-compose down
$ docker-compose -f app.yml down
$ rm terraform/terraform.tfstate
```

Or if you can use Makefile: `make clean`
