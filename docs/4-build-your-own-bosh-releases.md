# 4. Build Your Own BOSH Releases

This guide will walk you through the process of building your own BOSH releases for deploying services on a BOSH Director.

## 1. Create and Initialize a Release

You can create a BOSH release repository using either of the following approaches:

**Approach 1 (Single Command)**

```sh
bosh init-release --dir redis-release --git
```

This command creates the `redis-release` directory, initializes it for a BOSH release, and sets up Git for version control (this part is optional).

**Approach 2 (Multiple Steps)**

Alternatively, you can manually create the directory, navigate into it, initialize the release, and set up Git as separate steps:
```sh
mkdir redis-release 
cd redis-release 
bosh init release --git
```

This command creates the `redis-release` directory, initializes it for a BOSH release, and sets up Git for version control (this part is optional).

## 2. Add Packages

Packages are collections of software that your BOSH release will use. You need to define them in the `packages/` directory. Create a package by running:

```sh
bosh generate package <package-name>
```

For each package, you will need:

- A `packaging` file that describes how to build and compile the package.
- A `spec` file that lists the dependencies of the package.

Here is a simple example of what a `packaging` file might look like:
```sh
#!/bin/bash 
set -e 

# Example to install a package from a tarball 
tar xvf path-to-your-software.tar.gz -C /var/vcap/packages/package-name
```

Here is an example of a simple `spec` file for a package:
```sh
name: my-package 
dependencies: [] 
files: - my-files/*
```

Explanation of `spec` Fields:
- **`name`**: This is the name of the package. It should match the name of the package folder.
- **`dependencies`**: Lists other packages that this package depends on. These dependencies will be built and installed before this package.
- **`files`**: This specifies the source files needed for the package. You can list files from two main sources: **`src/` directory** and **Blobs**.

## 3. Add Blobs

To manage large files, such as binaries, you can add them to the `blobs/` directory using the following steps:

1. Add the binary file to the `blobs` directory:
```sh
bosh add-blob /path/to/local/file blobs/my-package/my-binary.tar.gz
```

2. Once you've added the blob, it will be available in your `blobs/` directory, and you can reference it in your package `spec` file.

## 4. Add Jobs

Jobs define processes that BOSH will run on the deployed VMs. To generate a job, run:

```bash
bosh generate job <job-name>
```

Each job has a few key components:
- **`monit` file**: Used to monitor and manage processes, ensuring the service is running and can be restarted if necessary.
- **`templates` directory**: Stores configuration files, scripts, or templates that will be used to configure the service on the VM. This is where the `ctl` (control) script is located.
- **`spec` file**: Defines properties the job can use. This includes settings like port numbers, file paths, or custom parameters that are passed from the deployment manifest.

**Example of `monit` File**

The `monit` file is responsible for starting, stopping, and monitoring your process. For example:
```bash
check process my-job 
with pidfile /var/vcap/sys/run/my-job/my-job.pid 
start program "/var/vcap/jobs/my-job/bin/start" 
stop program "/var/vcap/jobs/my-job/bin/stop"
```

**Example of `ctl` (Control) File**

The control (`ctl`) file defines how to start and stop your service. Here’s a simplified, generic version:
```bash
#!/bin/bash

JOB_NAME=my-job
BASE_DIR=/var/vcap
SYS_DIR=$BASE_DIR/sys
RUN_DIR=$SYS_DIR/run/$JOB_NAME
LOG_DIR=$SYS_DIR/log/$JOB_NAME
PIDFILE=$RUN_DIR/$JOB_NAME.pid
RUNAS=vcap

case $1 in

  start)
    mkdir -p $RUN_DIR $LOG_DIR
    chown -R $RUNAS:$RUNAS $RUN_DIR $LOG_DIR

    echo $$ > $PIDFILE

    # Start the application (replace this with your service start command)
    exec /path/to/your/application \
      1>>$LOG_DIR/stdout.log \
      2>>$LOG_DIR/stderr.log
    ;;

  stop)
    kill -9 `cat $PIDFILE`
    rm -f $PIDFILE
    ;;

  *)
    echo "Usage: ctl {start|stop}" ;;
esac
```

This script performs the following:
- **Start**: Creates the necessary directories, starts the application, and logs the output.
- **Stop**: Stops the process by using the PID stored in the `$PIDFILE`.
- **Other**: Displays usage instructions if the start/stop commands are not provided.

You can customize the `exec` command inside the `start` block to launch your specific service.

## 5. Configure Release

In the `config/final.yml` file, you can configure important settings like the name of your release, the version, and where the blobs (external packages and dependencies) will be stored.

Here’s an example configuration:
```yaml
name: my-bosh-release
version: 1.0.0

blobstore:
  provider: local
  options:
    blobstore_path: /tmp
```

**Explanation:**

- **`name`**: This is the name of your BOSH release. Make sure to use a clear and unique name that reflects your application or service.
- **`version`**: The version number for your release. This helps to differentiate between different release builds. You can manage versions manually or use BOSH to auto-increment them.
- **`blobstore`**: Defines where large binary files (blobs) are stored. In this case, the **local** provider is used, meaning that the files are stored locally on the machine. You can set the **blobstore_path** to any appropriate location on the filesystem.
    - **`provider`**: You can use different blobstore providers (e.g., `local`, `s3`, etc.). In this example, we're using the `local` provider.
    - **`blobstore_path`**: This specifies the directory path where blobs will be stored. In this case, it's configured to store blobs in `/tmp`.

## 6. Build and Release

You can create a development release to test the deployment process before making a final release.

```bash
bosh create release 
bosh upload release
```

This will create a release tarball and upload it to the BOSH Director for testing.

To create a final release (for production use):
```bash
bosh create release --final
```

## 7. Deploying the Release

Now that you have created your BOSH release, you can create a BOSH deployment manifest to deploy it. Here’s a basic example:

```bash
name: my-deployment

releases:
- name: my-bosh-release
  version: latest

stemcells:
- alias: default
  os: ubuntu-jammy
  version: latest

instance_groups:
- name: my-instance-group
  instances: 1
  jobs:
  - name: my-job
    release: my-bosh-release
  stemcell: default
  vm_type: default
  networks:
  - name: default
```

After you define your deployment manifest, run the following to deploy:
```bash
bosh -d my-deployment deploy manifest.yml
```

## 8. Testing

During development, it’s a good idea to test your release regularly. You can deploy, check logs, and inspect your running jobs using:
```bash
bosh -d <deployment-name> vms 
bosh -d <deployment-name> logs <instance-name> 
bosh -d <deployment-name> ssh <instance-name>
```

## 9. Examples and Useful Links

Example of a BOSH release for deploying a simple Spring Boot application: https://github.com/nhairlahovic-evoila/spring-boot-bosh-release


Useful Links:
- https://www.cloudfoundry.org/blog/create-lean-bosh-release
- https://mariash.github.io/learn-bosh/#create_release
- https://www.starkandwayne.com/blog/build-bosh-releases-faster-with-language-packs/index.html

