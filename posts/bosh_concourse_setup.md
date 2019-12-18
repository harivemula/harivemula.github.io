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

- Create Firewall Rules
```bash


```

- List the firewall rules

```bash
gcloud compute firewall-rules list --filter="network:hkv-vpc-1" --format="table(
                name,
                network,
                direction,
                priority,
                sourceRanges.list():label=SRC_RANGES,
                destinationRanges.list():label=DEST_RANGES,
                allowed[].map().firewall_rule().list():label=ALLOW,
                targetTags.list():label=TARGET_TAGS,
                disabled,
                description
            )"
 ```
