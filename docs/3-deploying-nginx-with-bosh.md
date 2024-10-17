# 3. Deploying NGINX with BOSH

## Summary

1. **Upload the NGINX release** to the BOSH Director either via URL or manual download.
2. **Upload the Ubuntu Jammy stemcell** to ensure the VMs can be created.
3. **Deploy the NGINX manifest** to configure the instance and run the NGINX web server.

Once everything is set up, you will have NGINX running on a BOSH-managed VM. You can access and manage it using BOSH commands.

**Note:** If you have multiple BOSH environments or need to specify a particular environment, replace `bosh-env` in the commands with the actual alias of your BOSH environment. If you only have one environment and it's set as the default, this step may not be necessary.

## 1. Upload the NGINX Release

BOSH uses **releases** to package software for deployment. The NGINX release can be uploaded either directly from a URL or by downloading and uploading it manually.

**Option 1: Upload Release from URL**

```sh
bosh -e bosh-env upload-release --sha1 59dbc1e8dd5f4c85cca18dce1d5b70f11f9ddfcd \   https://bosh.io/d/github.com/cloudfoundry-community/nginx-release?v=1.21.6
```

**Option 2: Manual Download and Upload**

If the direct upload doesn't work, you can manually download the release and upload it to BOSH:

```bash
# Download the release
curl -L https://bosh.io/d/github.com/cloudfoundry-community/nginx-release?v=1.21.6 --output nginx-release-1.21.6.tgz

# Upload the release to BOSH
bosh -e bosh-env upload-release ./nginx-release-1.21.6.tgz
```

**Check Installed Releases**

To ensure the NGINX release is uploaded:

```bash
bosh -e bosh-env releases
```

## 2. Upload the Stemcell

A **stemcell** is the base operating system image used to deploy VMs. You’ll need the **Ubuntu Jammy stemcell** for this NGINX deployment.

More information about available stemcells can be found [here](https://bosh.io/stemcells/).

**Option 1: Upload from URL**

You can directly upload the stemcell from a URL:

```bash
bosh -e bosh-env upload-stemcell --sha1 5b80ef006507b5a318c40eb797c80824f77ae00b \
https://bosh.io/d/stemcells/bosh-vsphere-esxi-ubuntu-jammy-go_agent?v=1.555
```

**Option 2: Manual Download and Upload**

If direct upload doesn’t work, you can manually download and upload the stemcell:

```bash
# Download the stemcell
curl -L https://bosh.io/d/stemcells/bosh-vsphere-esxi-ubuntu-jammy-go_agent?v=1.555 --output bosh-vsphere-esxi-ubuntu-jammy.tgz

# Upload the stemcell to BOSH
bosh -e bosh-env upload-stemcell ./bosh-vsphere-esxi-ubuntu-jammy.tgz
```

To ensure the Ubuntu Jammy stemcell is uploaded:

```bash
bosh -e bosh-env stemcells
```


## 3. Deploy the NGINX Manifest

BOSH uses a **manifest** to describe how to deploy software. Below is an example manifest for deploying NGINX.

Save the following content as `nginx-deployment.yml` on your local machine:

```yml
---
name: nginx

releases:
- name: nginx
  version: latest

stemcells:
- alias: ubuntu
  os: ubuntu-jammy
  version: latest

instance_groups:
- name: nginx
  instances: 1
  azs: [ z1 ]
  vm_type: default
  persistent_disk_type: default
  stemcell: ubuntu
  networks:
  - name: default
  jobs:
  - name: nginx
    release: nginx
    properties:
      nginx_conf: |
        user nobody vcap;
        worker_processes  1;
        error_log /var/vcap/sys/log/nginx/error.log info;
        events {
          worker_connections  1024;
        }
        http {
          include /var/vcap/packages/nginx/conf/mime.types;
          default_type  application/octet-stream;
          sendfile on;
          ssi on;
          keepalive_timeout  65;
          server_names_hash_bucket_size 64;
          server {
            server_name _;
            listen *:80;
            access_log /var/vcap/sys/log/nginx/example.com-access.log;
            error_log /var/vcap/sys/log/nginx/example.com-error.log;
            root /var/vcap/data/nginx/document_root;
            index index.shtml;
          }
        }
      pre_start: |
        #!/bin/bash -ex
        NGINX_DIR=/var/vcap/data/nginx/document_root
        if [ ! -d $NGINX_DIR ]; then
          mkdir -p $NGINX_DIR
          cd $NGINX_DIR
          cat > index.shtml <<EOF
            <html><head><title>BOSH on IPv6</title></head><body>
            <h2>Welcome to BOSH's nginx Release</h2>
            <h2>My hostname/IP: <b><!--# echo var="HTTP_HOST" --></b><br />
            Your IP: <b><!--# echo var="REMOTE_ADDR" --></b></h2>
            </body></html>
        EOF
        fi
update:
  canaries: 1
  max_in_flight: 1
  serial: false
  canary_watch_time: 1000-60000
  update_watch_time: 1000-60000
```

Once you have saved the manifest, deploy it using the following command:

```bash
bosh -e bosh-env -d nginx deploy nginx-deployment.yml
```

- `bosh-env`: Replace with the actual environment alias.
- `nginx`: This is the deployment name.
- `nginx-deployment.yml`: Path to the manifest file you saved.


**Check VMs and IP Address**

After deploying, you can check the status of your VMs and ensure that NGINX is running:
```bash
bosh -e bosh-env -d nginx vms
```

This command will show the VMs created as part of the deployment. You can also see the IP address to access the deployed NGINX instance (e.g., `10.244.0.34`).

**SSH into the VM**

To verify the deployment, you can also SSH into the VM and test if NGINX is running.
```sh
bosh -e bosh-env -d <deployment-name> ssh
```

Once inside the VM, you can test if NGINX is running:
```bash
curl http://localhost
```

If NGINX is running, you should see the default NGINX page or a custom page set up in the deployment.

**Delete Deployment**

```bash
bosh -e bosh-env -d nginx delete-deployment
```