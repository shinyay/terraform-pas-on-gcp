# terraform-pas-on-gcp

---

## Notes
### Pivnet
#### Terraform Template - PAS
```
$ pivnet releases -p elastic-runtime
$ pivnet product-files -p elastic-runtime -r 2.4.2
$ pivnet download-product-files -p elastic-runtime -r 2.4.2 -i 277283
```

#### Ops Manager URL
```
$ pivnet releases -p ops-manager
$ pivnet product-files -p ops-manager -r 2.4.3
$ pivnet download-product-files -p ops-manager -r 2.4.3 -i 302870
```

### BOSH Director on GCP
#### Director Config
- NTP Servers
  - metadata.google.internal
- Enable VM Resurrector Plugin
  - Enabled
- Enable Post Deploy Scripts
  - Enabled
- Recreate all VMs
  - Enabled
- Recreate All Persistent Disks
  - Enabled
- Enable bosh deploy retries
  - Enabled
- Keep Unreachable Director VMs
  - Enabled

#### Create Availability Zones
- Google Availability Zone
  - **Terraform output** : `azs`
    - asia-northeast1-a
    - asia-northeast1-b
    - asia-northeast1-c

#### Create Networks
- management
  - `network_name`/`management_subnet_name`/`region`
    - pcf-pcf-network/pcf-infrastructure-subnet/asia-northeast1
  - `management_subnet_cidrs`
    - 10.0.0.0/26
  - Reserved IP Ranges
    - 10.0.0.1-10.0.0.9
  - DNS
    - 169.254.169.254
  - Gateway
    - 10.0.0.1

- pas
  - `network_name`/`pas_subnet_name`/`region`
    - pcf-pcf-network/pcf-pas-subnet/asia-northeast1
  - `pas_subnet_cidrs`
    - 10.0.4.0/24
  - Reserved IP Ranges
    - 10.0.4.1-10.0.4.9
  - DNS
    - 169.254.169.254
  - Gateway
    - 10.0.4.1

- services
  - `network_name`/`services_subnet_name`/`region`
    - pcf-pcf-network/pcf-services-subnet/asia-northeast1
  - `services_subnet_cidrs`
    - 10.0.8.0/24
  - Reserved IP Ranges
    - 10.0.8.1-10.0.8.9
  - DNS
    - 169.254.169.254
  - Gateway
    - 10.0.0.1

#### Assign AZs and Networks
- asia-northeast1-a
- management

#### Resource Config
- BOSH Director
  - large.disk
- Mater Compilation Job
  - medium

### SSH to OPS Manager VM
```
$ gcloud compute --project $GCP_PROJECT ssh --zone $ZONE pcf-ops-manager
```

```
$ wget https://github.com/pivotal-cf/om/releases/download/0.44.0/om-linux
$ wget https://github.com/pivotal-cf/pivnet-cli/releases/download/v0.0.55/pivnet-linux-amd64-0.0.55
$ sudo mv om-linux /usr/local/bin/om
$ sudo mv pivnet-linux* /usr/local/bin/pivnet
$ sudo chmod +x /usr/local/bin/om
$ sudo chmod +x /usr/local/bin/pivnet
```

```
$ pivnet login --api-token=$TOKEN
$ pivnet download-product-files -p elastic-runtime -r $RELEASE -i $ID
```

```
$ om --target https://localhost -k -u admin -p admin --request-timeout 3600 upload-product -p cf-${VERSION}.pivotal
$ om --target https://localhost -k -u admin -p admin stage-product -p cf -v $VERSION
```

### PAS
#### AZ and Network Assignments
- Network
  - pas

#### Domains
- System Domain
  - `sys_domain`
  - sys.pcf.$DOMAIN
- Apps Domain
  - 
  - apps.pcf.$DOMAIN

#### Networking
- Certificates and Private Keys for HAProxy and Router
  - pas
  - Generate RSA Certificate
    - *.pcf.$DOMAIN,*.sys.pcf.$DOMAIN,*.login.sys.pcf.$DOMAIN,*.uaa.sys.pcf.$DOMAIN,*.apps.pcf.$DOMAIN
- HAProxy forwards requests to Router over TLS
  - Disabled
- Disable SSL certificate verification for this environment
  - Enabled

#### UAA
- SAML Service Provider Credentials
  - *.pcf.$DOMAIN,*.sys.pcf.$DOMAIN,*.login.sys.pcf.$DOMAIN,*.uaa.sys.pcf.$DOMAIN,*.apps.pcf.$DOMAIN

#### CredHub
- Encryption Key
  - Primary

#### Resource Config
- Router
  - `ws_router_pool`
  - `http_lb_backend_name`
    - tcp:pcf-cf-ws,http:pcf-httpslb
- Diego Brain
  - `ssh_router_pool`
    - tcp:pcf-cf-ssh

### PAS VMs Stop and Start
1. Access to `BOSH Director tile` and select the `Status` tab to get **IP ADDRESS**
2. Select the `Credentials` tab and get `Director Credentials`
3. `$ gcloud compute --project $GCP_PROJECT ssh --zone $ZONE pcf-ops-manager`
4. `$ sudo su - ubuntu`
5. `$ bosh alias-env gcp -e $DIRECTOR-IP-ADDRESS --ca-cert /var/tempest/workspaces/default/root_ca_certificate`
6. `$ bosh -e gcp log-in`
7. `$ bosh -e gcp vms`
8. `$ bosh -e gcp -d $DEPLOYMENT stop --hard`

#### Check
1. `$ bosh -e gcp vms`
2. `$ bosh -e gcp -d $DEPLOYMENT deployment`

#### Start
1. `$ find /var/tempest/workspaces/default/deployments -name cf-*.yml`
2. `$ bosh -e gcp -d $DEPLOYMENT start`

### SSH to OPS Manager VM

- Check VM asigned Zonw
```
$ gcloud compute instances list
```

- SSH to OPSMAN VM
```
$ gcloud compute ssh ubuntu@pcf-ops-manager --zone <ZONE> --force-key-file-overwrite --strict-host-key-checking=no
```

```
ubuntu@pcf-ops-manager:~$ export REFRESH_TOKEN=<YOUR_REFRESH_TOKEN>
ubuntu@pcf-ops-manager:~$ ubuntu@pcf-ops-manager:~$ curl -s https://network.pivotal.io/api/v2/authentication/access_tokens -d "{\"refresh_token\":\"${REFRESH_TOKEN}\"}" | jq -r .access_token
export ACCESS_TOKEN=<YOUR_ACCESS_TOKEN>
```

```
ubuntu@pcf-ops-manager:~$ export FILENAME=cf-2.1.12-build.1.pivotal
ubuntu@pcf-ops-manager:~$ export DOWNLOAD_URL=https://network.pivotal.io/api/v2/products/elastic-runtime/releases/162824/product_files/195393/download
```

```
ubuntu@pcf-ops-manager:~$ wget -O ${FILENAME} --header='Authorization: Bearer ${ACCESS_TOKEN}' ${DOWNLOAD_URL}
```

```
ubuntu@pcf-ops-manager:~$ wwget -O om https://github.com/pivotal-cf/om/releases/download/0.38.0/om-linux && chmod +x om && sudo mv om /usr/local/bin/
```

```
ubuntu@pcf-ops-manager:~$ export PRODUCT_NAME=`basename $FILENAME .pivotal | python -c 'print("-".join(raw_input().split("-")[:-2]))'`
ubuntu@pcf-ops-manager:~$ export PRODUCT_VERSION=(`basename $FILENAME .pivotal | python -c 'print("-".join(raw_input().split("-")[-2:]))'`)
ubuntu@pcf-ops-manager:~$ om --target https://localhost -k -u admin -p admin --request-timeout 3600 upload-product -p ~/$FILENAME
ubuntu@pcf-ops-manager:~$ om --target https://localhost -k -u admin -p admin stage-product -p $PRODUCT_NAME -v $PRODUCT_VERSION
```

```
ubuntu@pcf-ops-manager:~$ export SC_FILENAME=light-bosh-stemcell-3541.44-google-kvm-ubuntu-trusty-go_agent.tgz
ubuntu@pcf-ops-manager:~$ export SC_DOWNLOAD_URL=https://network.pivotal.io/api/v2/products/stemcells/releases/162201/product_files/194935/download

```

```
ubuntu@pcf-ops-manager:~$ wget -O ${SC_FILENAME} --header='Authorization: Bearer ${ACCESS_TOKEN}' ${SC_DOWNLOAD_URL}
ubuntu@pcf-ops-manager:~$ om --target https://localhost -k -u admin -p admin --request-timeout 3600 upload-stemcell -s ~/${SC_FILENAME}
```

### Create Networks
- Name: management
- Google Network Name: pcf-pcf-network/pcf-management-subnet/asia-northeast1
- CIDR: 10.0.0.0/26
- RANGE: 10.0.0.1-10.0.0.9
- DNS: 169.254.169.254
- Gateway: 10.0.0.1

- Name: pas
- Google Network Name: pcf-pcf-network/pcf-pas-subnet/asia-northeast1
- CIDR: 10.0.4.0/24
- RANGE: 10.0.4.1-10.0.4.9
- DNS: 169.254.169.254
- Gateway: 10.0.4.1

- Name: services
- Google Network Name: pcf-pcf-network/pcf-services-subnet/asia-northeast1
- CIDR: 10.0.8.0/24
- RANGE: 10.0.8.1-10.0.8.9
- DNS: 169.254.169.254
- Gateway: 10.0.8.1

### Stop and Start with BOSH
```
$ bosh2 alias-env gcp -e 10.0.0.10 --ca-cert /var/tempest/workspaces/default/root_ca_certificate
$ bosh -e gcp log-in
$ bosh -e gcp vms
$ bosh -e gcp -d <DEPLOYMENT_NAME> stop --hard
$ find /var/tempest/workspaces/default/deployments -name cf-*.yml
$ bosh -e gcp -d <DEPLOYMENT_NAME> start
```
