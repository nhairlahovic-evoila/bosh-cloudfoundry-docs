# 2. Basic BOSH Operations

**Note:** Replace `bosh-env` with the actual name or alias of your BOSH environment.

### 1. **List Deployments**

To list all the deployments managed by the BOSH Director:

```sh 
bosh -e bosh-env deployments
```

### 2. **Upload a Stemcell**

To upload an operating system image (stemcell) to the BOSH Director:

```sh 
bosh -e bosh-env upload-stemcell <stemcell_url>
```

### 3. **Upload a Release**

**1. Upload a Release from a URL**

To upload a release directly from a URL (such as from [bosh.io](https://bosh.io/releases/)):

```sh 
bosh -e bosh-env upload-release <release_url>
```

**2. Upload a Release from a `.tgz` File**

If you have already downloaded the release as a `.tgz` file (for example, using `curl` or `wget`), you can upload it to BOSH from your local machine:

```sh 
bosh -e bosh-env upload-release </path/to/release-file>.tgz
```

### 4. **Deploy Software**

To deploy software using a deployment manifest:

```sh 
bosh -e bosh-env -d <deployment_name> deploy <manifest_file.yml>
```

### 5. **Delete a Deployment**

To delete a deployment and remove the associated VMs:

```sh 
bosh -e bosh-env -d <deployment_name> delete-deployment
```