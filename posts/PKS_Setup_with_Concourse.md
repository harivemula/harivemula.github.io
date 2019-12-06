

# Setup PKS Using Concourse in GCP


## Setup the Pre-requisites in GCP using Terraform
Create a folder with name 'workspace' in user home, clone the [repository](https://github.com/pivotal-cf/terraforming-gcp).
Change the directory to _terraforming-pks_ and create a file `terraform.tfvars` with the below content.
```
env_name         = "ci-pks"
opsman_image_url = ""
region           = "asia-southeast1"
zones            = ["asia-southeast1-b"]
project          = "pa-hvemula"
dns_suffix       = "env1.k8scloud.cf"
service_account_key = "terraform-svc.json"
create_gcs_buckets = true
external_database = false
```

```
terraform init
terraform plan -out=plan
terraform apply plan
terraform output
terraform output -j | jq -r .
```

## Generate Pivnet token
- Login to network.pivotal.io
- Under user name -> Click on Edit Profile
- Click on 'Request New Refresh Token' to generate a pivnet token.

## Configure Credhub
- Set Credhub target
```
credhub api --ca-cert <credhub-ca.pem> https://<credhub-ip>:8844
```
- Login to credhub using client secret
```
credhub login --client-name=credhub-admin --client-secret=<client secret>
```

- Set the below cedentials in Credhub under your team name (as per concourse pipeline)
Pivnet Token
```
credhub set -n /concourse/team-a/pivnet-refresh-token -t value -v <token>
credhub set -n /concourse/team-a/pivnet-token -t value -v <token>
```
- Verify credhub element, and ensure it created properly.
```
credhub get -n /concourse/team-a/pivnet-refresh-token
```

### Set the credhub entries for below under /team-a/

#### Provide Below from terraform output

- `gcp_service_account_json`: GCP Service Account JSON - Same as the one you have used to run terraform.
```
cat <serivceaccount-token.json>|jq -c
credhub set -n /concourse/team-a/gcp_service_account_json -t value -v '<json output of above command>'
```

- `gcp_opsman_sa_json`: SA JSON content of the opsman service account
```
credhub set -n /concourse/team-a/gcp_opsman_sa_json -t value -v "$(terraform output -json | jq -r .opsman_service_account_key.value)"
```

- `opsman_public_ip`: Value from 'ops_manager_public_ip' of terraform output
```
credhub set -n /concourse/team-a/opsman_public_ip -t value -v "$(terraform output -json | jq -r .ops_manager_public_ip.value)"
```

- `gcp_pks_master_sa_json`: value from 'pks_master_node_service_account_key'
```
credhub set -n /concourse/team-a/gcp_pks_master_sa_json -t value -v "$(terraform output -json | jq -r .pks_master_node_service_account_key.value | jq -c)"
```

- `opsman-ssh-pub`: value from 'ops_manager_ssh_public_key'
```
crehub set -n $CREDHUB_PREFIX/opsman-ssh-pub -t value -v "$(terraform output -json | jq -r .ops_manager_ssh_public_key.value)"
```

- `gcp-project-id`: Your GCP Project Id
```
credhub set -n /concourse/team-a/gcp-project-id -t value -v "$(terraform output -json | jq -r .project.value)"
```

- `gcp-pks-master-sa` : Master Node(s) service account email.
```
credhub set -n /concourse/team-a/gcp-pks-master-sa -t value -v "$(terraform output -json | jq -r .pks_master_node_service_account_email.value)"
```

- `gcp-pks-worker-sa`: Worker node(s) service account email.
```
credhub set -n /concourse/team-a/gcp-pks-worker-sa -t value -v "$(terraform output -json | jq -r .pks_worker_node_service_account_email.value)"
```

- `opsman-url`: Ops Manager endpoint domain name
```
credhub set -n /concourse/team-a/opsman-url -t value -v "https://$(terraform output -json | jq -r .ops_manager_dns.value)"
```

- `pks-api-fqdn` : PKS API Endpoint domain name
```
credhub set -n /concourse/team-a/pks-api-fqdn -t value -v "$(terraform output -json | jq -r .pks_api_endpoint.value)"
```


#### Manually provide the values for below.


- `opsmanager_decryption_password`: Ops manager decryption passphrase.
- `opsmanager_admin_password`: Set a new password for opsman admin user
- `opsmanager_admin_user`: Set a new user name for opsman.
- `credhub-ca-cert`:  Credhub ca cert
```
credhub set -n /concourse/team-a/domain-crt -t value -v "$(cat credhub-ca.pem)"
```
- `credhub-secret`: Credhub client secret
- `credhub-client`: Credhub client name
- `plat-auto-pipes-deploy-key`: Your own ssh keys, in next steps you will configure your git to use this key. So that concourse pipeline can clone or commit changes to your repo.
```
credhub set -n /concourse/team-a/plat-auto-pipes-deploy-key -t ssh -p ~/.ssh/id_rsa -u ~/.ssh/id_rsa.pub
```

- `pks_tls`: Generate Certs for your domain name in terraform output for `pks_api_endpoint` (wildcard).
   Refer the [link] (#self-signed-cert-generation) to generate your own ca and certs.

```
credhub set -n /concourse/team-a/pks_tls -t certificate -c <pks.cert> -p <pks.key>
```
- `domain-crt`: Your own root ca cert
```
credhub set -n /concourse/team-a/domain-crt -t value -v "$(cat hari_rootca.crt)"
```

## Configure github
- Fork the [repo](https://github.com/harivemula/platform-automation-pks)

### Configure github for ssh keys
- Go to Settings under your name
- Go to SSH and GPG keys
- Click on 'Add New' under SSH keys
- Give a name to remember this key and paste public key which was used for property `plat-auto-pipes-deploy-key`.


## Working on pipeline
- Clone the repo that you forked above.

Ex:
```
git clone https://github.com/harivemula/platform-automation-pks
```
- Change directory to platform-automation-pks
- Delete the content of state.yml
- Open vars.yml
  - `opsman-url`: is from your terraform output - key: ops_manager_dns
  - `pipeline-repo`: get the repo ssh url from github
  - `subdomain-name`: Give the env_name value from your terraform.tfvars
  - `region`: Region you have mentioned in terraform.tfvars
  - `az1`: AZ Name from your terraform.tfvars
  - `gcs_bucket`: GCS Bucket name - to store the backups of foundation.

- Open env.yml
  - `target`: set the opsman url from terraform output.



## Login to Concourse using Fly

```
fly -t ci login -u <username> -p <password> -n <team-a> -c <concourse url | http://<ip>:8080>
```
