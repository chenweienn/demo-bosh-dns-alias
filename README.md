# demo-bosh-dns-alias

## Pre-requisition

* [BOSH DNS alias](https://bosh.io/docs/dns/#aliases)

* [BOSH runtime-config](https://bosh.io/docs/terminology/#addon)

## Procedure

1. upload [os-conf release](https://bosh.io/releases/github.com/cloudfoundry/os-conf-release?all=1) to BOSH director

```
bosh upload-release --sha1 7ef05f6f3ebc03f59ad8131851dbd1abd1ab3663 \
  https://bosh.io/d/github.com/cloudfoundry/os-conf-release?v=22.1.0
```
2. Prepare test-dns-alias.yml

```
releases:
- name: os-conf
  version: 22.1.0
 
addons:
- name: test-dns-alias
  jobs:
  - name: pre-start-script
    release: os-conf
    properties:
      script: |-
        #!/bin/bash
        mkdir -p /var/vcap/jobs/test-dns-alias/dns/
        cat >  /var/vcap/jobs/test-dns-alias/dns/aliases.json << EOF
        {  "testingdomain":["1.2.3.4"],
           "_.sys.DOMAIN":["10.36.93.250"],
           "_.apps.DOMAIN":["10.36.93.250"],
           "_.tcp.DOMAIN":["10.36.93.249"],
           "ssh.DOMAIN":["10.36.93.248"],
           "opsmgr.DOMAIN":["10.36.93.247"]
        }
        EOF
  include:
    instance_groups:
    - diego_cell
```
The optional `include` section is used to control placement rules of the addon.

3. update the runtime-config into BOSH director

```
bosh update-runtime-config --name=test-dns-alias ./test-dns-alias.yml
bosh configs
```

4. deploy the cf 

```
CF_DEP=$(bosh ds --column name | grep cf- | cut -f 1)
bosh -d ${CF_DEP} manifest > ${CF_DEP}.yml
bosh -d ${CF_DEP} deploy ${CF_DEP}.yml
```

5. Once the deployment completes successfully, you could verify that the runtime-config **test-dns-alias** has been associated with the TAS deployment

```
bosh curl "/deployment_configs?deployment[]=${CF_DEP}" | jq .
```

6. test if an app can resolve testingdomain to 1.2.3.4

```
cf ssh APP
nslookup testingdomain
```
