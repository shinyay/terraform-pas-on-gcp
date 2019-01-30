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
