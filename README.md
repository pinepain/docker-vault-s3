# Vault S3

This is a [Hashicorp Vault](https://www.vaultproject.io/) image preconfigured for S3 backend.

## Prerequisites

You will need to have preconfigure S3 bucket and IAM user which can access this bucket.

It might be a good idea to enable bucket versioning.

Having separate IAM user for vault bucket is also nice. The user needs to have following permissions:

```json
{
       "Version": "2012-10-17",
       "Statement": [
           {
               "Effect": "Allow",
               "Action": [
                   "s3:ListBucket"
               ],
               "Resource": [
                   "<bucket full arn>"
               ]
           },
           {
               "Effect": "Allow",
               "Action": [
                   "s3:PutObject",
                   "s3:GetObject",
                   "s3:DeleteObject"
               ],
               "Resource": [
                   "<bucket full arn>/*"
               ]
           }
       ]
   }
```

## Usage

Export AWS credentials

```
export AWS_ACCESS_KEY_ID="<aws access key id>"
export AWS_SECRET_ACCESS_KEY="<aws secret access key>"
export AWS_S3_BUCKET="<aws s3 bucket>"
export AWS_DEFAULT_REGION="<aws region>"
```

Run image:

`docker run -p 8200:8200 --cap-add IPC_LOCK -e AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY -e AWS_S3_BUCKET -e AWS_REGION -d pinepain/vault`

and export vault address:

`export VAULT_ADDR=http://127.0.0.1:8200`

You'll get container id. To view logs you can run `docker logs <conatiner id>`

```
==> Vault server configuration:

                     Cgo: disabled
              Listener 1: tcp (addr: "0.0.0.0:8200", cluster address: "0.0.0.0:8201", tls: "disabled")
               Log Level: info
                   Mlock: supported: true, enabled: true
                 Storage: s3
                 Version: Vault v0.9.0
             Version Sha: bdac1854478538052ba5b7ec9a9ec688d35a3335

==> Vault server started! Log data will stream in below:
```

Note, that in production you will probably want to run vault container daemonized and keep it up and running forever:

`docker run -p 8200:8200 --restart=always --cap-add IPC_LOCK -e AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY -e AWS_S3_BUCKET -e AWS_REGION -d pinepain/vault`


## Initializing vault

*Skip this section if you have vault data initialized previously* 

If you haven't run vault with this backend before, you'll need to initialize it first. This is quite simple, just run
`vault init`. By default, you should get 5 unseal keys and one root token. Store them in a secure way as there is little
no chance to restore them and you won't get you data from vault without them.

## Unsealing

By default vault is in sealed state whenever it starts, including restarts, so you'll need to unseal it first. By default
you need 3 unseal keys to unseal, so run `vault unseal` three times and provide three unseal keys on by one.

Now you should be good to go, `vault status` should gives you something like

```
vault status
Type: shamir
Sealed: false
Key Shares: 5
Key Threshold: 3
Unseal Progress: 0
Unseal Nonce: 
Version: 0.9.0
Cluster Name: vault-cluster-<hash>
Cluster ID: <another hash>

High-Availability Enabled: false
```

At this point you probably want to start using vault so you need to export your token, so run `VAULT_TOKEN=<root token>`.

Cool. Let's try to fill first secret. Vault has [nice intro](https://www.vaultproject.io/intro/getting-started/first-secret.html),
here's it short version:

```
$ vault write secret/hello value=world
Success! Data written to: secret/hello

$ vault read secret/hello
Key             	Value
---             	-----
refresh_interval	768h0m0s
value           	world

$ vault write secret/hello value=world excited=yes
Success! Data written to: secret/hello

$ vault read secret/hello
Key             	Value
---             	-----
refresh_interval	768h0m0s
excited         	yes
value           	world

$ vault read -format=json secret/hello
{
	"request_id": "aae87106-f258-8828-fd34-d03b9ffa6be6",
	"lease_id": "",
	"lease_duration": 2764800,
	"renewable": false,
	"data": {
		"excited": "yes",
		"value": "world"
	},
	"warnings": null
}

$ vault delete secret/hello
Success! Deleted 'secret/hello' if it existed.
```
