# Setup Bosh & Concourse in GCP

## Create Pre-requisites in GCP
- Create a network
```bash
gcloud compute networks create vpc-bosh-concourse --subnet-mode custom
```
- Create subnet
```bash
gcloud compute networks subnets create subnet-bosh-concourse --network vpc-bosh-concourse --range 10.1.0.0/24 --enable-private-ip-google-access
```
