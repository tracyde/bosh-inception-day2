# Bosh Inception - Day 1

bosh-inception-day2

https://github.com/tracyde/bosh-inception-day2.git

---

# Agenda

1. Environment Recap
1. Lab - Bosh director check
1. What is a Bosh Release
1. What are we going to deploy
1. Lab - Create release skeleton
1. Determine dependencies
1. Lab - Add dependencies to release
1. Lab - Create job
1. Lab - Update monit files
1. Lab - Update job specs
1. Lab - Create Package Skeletons
1. Lab - Update packaging specs
1. Lab - Create packaging scripts
1. Lab - Update job specs with dependencies
1. Add Blobs
1. Lab - Configure a blobstore
1. Lab - Add Blobs
1. Lab - Create Job Properties
1. Lab - Create a Dev Release
1. Lab - Create Release Manifest
1. Lab - Deploy Release

---

# Environment Recap

What your AWS environment should look like

.image-100[![Image of Environment](./images/environment.png)]

---

# Lab - Bosh director check

* ssh into bastion host (jumpbox)
* target bosh director
 - `bosh target 10.10.1.6`
* If you can not target director, deploy one below
* deploy bosh director
 - `cd ~/workspace/bosh-inception-day1/deployments`
 - edit `bosh.yml`
   - __bosh_subnet__
   - __availability_zone__
   - __region__
   - __s3_region__
 - `bosh-init deploy bosh.yml`
 - `bosh target 10.10.1.6`

---

# What is a Bosh Release

> A release is a __versioned__ collection of configuration properties, configuration templates, start up scripts, source code, binary artifacts, and anything else required to build and deploy software in a __reproducible__ way.

---

# What is a Bosh Release

A release contains one or more pieces of software that work together. For example, you could create a release of a service with three pieces: two MySQL nodes and a dashboard app.

There are four fundamental elements in a release:

--

* __Jobs__ describe pieces of the service or application you are releasing
--

* __Packages__ provide source code and dependencies to jobs
--

* __Source__ provides packages the non-binary files they need
--

* __Blobs__ provide packages the binaries they need, other than binaries that are checked into a source code repository

---

# What are we going to deploy

--

## Gogs - [Go Git Service](https://gogs.io/)
.image-100[![Gogs Screenshot](./images/gogs-screenshot.png)]

---

# Lab - Create release skeleton

_Use dashes in the release name. Use underscores for all other filenames in the release._

```
cd ~/workspace
bosh init release gogs-release --git
mv ~/workspace/.gitignore ~/workspace/gogs-release
```

---

# Lab - Create release skeleton

When deploying your release, BOSH places compiled code and other resources in the /var/vcap/ directory tree, which BOSH creates on the job VMs.

--

The four directories you just created, `jobs`, `packages`, `src`, and `blobs`, appear on job VMs as `/var/vcap/jobs`, `/var/vcap/packages`, `/var/vcap/src`, and `/var/vcap/blobs`, respectively.

The `config` directory stores the information BOSH needs about the blobstore. More on this later.
---

# Determine dependencies

This takes some time, must look through the application sourcecode to determine external dependencies (aka libraries) that are required during compile time. Remember that bosh compiles the package from scratch and can be initiated in environments that do not have an internet connection.

The dependencies go into your releases `src` directory and can be either copies of the sourcecode or git sub-modules.

---

# Lab - Add dependencies to release

We are going to find and add the dependencies for Gogs.
This is a bit easier since gogs is a go application.

First we will install go to `/usr/local`

```
cd ~
wget https://storage.googleapis.com/golang/go1.6.3.linux-amd64.tar.gz
sudo tar xvf go1.6.3.linux-amd64.tar.gz
sudo mv go /usr/local/
export PATH=$PATH:/usr/local/go/bin
go version
```

Then we are going to create a directory to download all of gogs dependencies and use the `go get` command to download all dependencies required by gogs.

```
mkdir ~/workspace/gogs
export GOPATH=~/workspace/gogs
go get -u -tags "sqlite cert" github.com/gogits/gogs
cd ~/workspace/gogs
cp -R src/* ~/workspace/gogs-release/src/
cd ~/workspace/gogs-release
```

---

# Lab - Create job

Generate the job structure

```
cd ~/workspace/gogs-release
bosh generate job gogs_web
```

---

# Lab - Create job

Create control scripts

Every job needs a way to start and stop. You provide that by writing a control script and updating the `monit` file.

The control script:
* Includes a start command and a stop command.
* Is an ERB template stored in the templates directory for the relevant job.

For the `gogs_web` job, create a control script that configures the job to store logs in `/var/vcap/sys/log/JOB_NAME`. Save this script as `ctl.erb` in the `templates` directory for its job.

---

# Lab - Create job

The control script for the `gogs_web` job looks like this:

```
#!/bin/bash

RUN_DIR=/var/vcap/sys/run/gogs_web
LOG_DIR=/var/vcap/sys/log/gogs_web
DATA_DIR=/var/vcap/store/gogs_web
PIDFILE=${RUN_DIR}/pid

case $1 in

  start)
    mkdir -p $RUN_DIR $LOG_DIR $DATA_DIR
    chown -R vcap:vcap $RUN_DIR $LOG_DIR $DATA_DIR

    echo $$ > $PIDFILE

    cd /var/vcap/packages/gogs

    if [ ! -e /var/vcap/store/gogs_web/app.ini ]; then
      cp /var/vcap/jobs/gogs_web/app.ini /var/vcap/store/gogs_web/app.ini
    fi

    USER=vcap HOME=${DATA_DIR} exec /var/vcap/packages/gogs/bin/gogs web --config /var/vcap/store/gogs_web/app.ini \
      >>  $LOG_DIR/gogs_web.stdout.log \
      2>> $LOG_DIR/gogs_web.stderr.log

    ;;

  stop)
    kill -9 `cat $PIDFILE`
    rm -f $PIDFILE

    ;;

  *)
    echo "Usage: ctl {start|stop}" ;;

esac
```

---

# Lab - Create job

For the `gogs_web` job, we need a configuration file that contains configuration info for gogs. Save this file as `app.ini.erb` in the `templates` directory for its job.

The config file for the `gogs_web` job looks like this:

```
[repository]
ROOT = /var/vcap/store/gogs_web/repositories
RUN_USER = vcap
```

---

# Lab - Update monit files

The `monit` file:
* Specifies the process ID (pid) file for the job
* References each command provided by the templates for the job
* Specifies that the job belongs to the `vcap` group

The `monit` file for the `gogs_web` job looks like this:

```
check process gogs_web
  with pidfile /var/vcap/sys/run/gogs_web/pid
  start program "/var/vcap/jobs/gogs_web/bin/ctl start"
  stop program "/var/vcap/jobs/gogs_web/bin/ctl stop"
  group vcap
```

---

# Lab - Update job specs

At compile time, BOSH transforms templates into files, which it then replicates on the job VMs.

The template names and file paths are among the metadata for each job that resides in the job `spec` file.

In the job `spec` file, the `templates` block contains key/value pairs where:
* Each key is template name
* Each value is the path to the corresponding file on a job VM

The file paths that you provide for templates are relative to the `/var/vcap/jobs/<job_name>` directory on the VM. For example, `bin/ctl` becomes `/var/vcap/jobs/<job_name>/bin/ctl` on the job VM. Using bin as the directory where these files go is a convention.

The `templates` block of the updated `spec` files for the example jobs look like this:

```
templates:
  ctl.erb: bin/ctl
  app.ini.erb: app.ini
```

For the `gogs_web` job, update the `spec` file with template names.

--

## Commit

You have now created your job skeletons; this is a good time to commit!

```
cd ~/workspace/gogs-release
git add .
git commit -m 'created gogs_web job'
```

---

# Lab - Create Package Skeletons

Packages give BOSH the information needed to prepare the binaries and dependencies for your jobs.

```
bosh generate package golang1.6
bosh generate package gogs
```

Putting each dependency in a separate package provides maximum reusability along with a clear, modular structure.

_Note: Use of the pre_packaging file is not recommended, and is not discussed in this tutorial._

---

# Lab - Update packaging specs

Within each package directory, there is a `spec` file which states:
* The package name
* The package’s dependencies
* The location where BOSH can find the binaries and other files that the package needs at compile time

To maximize portability of your release across different versions of stemcells, never depend on the presence of libraries or other software on stemcells.

BOSH interprets the locations you record in the files section as being either in the `src` directory or in the `blobs` directory. (BOSH looks in `src` first.) When you add the actual blobs to a blobstore (see the next section), BOSH populates the `blobs` directory with the correct information.

## golang package spec

```
---
name: golang1.6

files:
  - golang/go1.6.3.linux-amd64.tar.gz
```

_For packages that depend on their own source code, use the globbing pattern `<package_name>/**/*` to deep-traverse the directory in `src` where the source code should reside._

## gogs package spec

```
---
name: gogs

dependencies:
  - golang1.6

files:
  - github.com/Unknwon/cae/tz/testdata/test.lnk
```

---

# Lab - Create packaging scripts

At compile time, BOSH takes the source files referenced in the package specs, and renders them into the executable binaries and scripts that your deployed jobs need.

You write packaging scripts to instruct BOSH how to do this. The instructions may involve some combination of copying, compilation, and related procedures.

BOSH relies on you to write packaging scripts that perform the correct operation.

---

# Lab - Create packaging scripts

Adhere to these principles when writing packaging scripts:

* Use your dependency graph to determine which dependencies belong in each packaging script.

* Begin each script with a `set -e -x` line. This aids debugging at compile time by causing the script to exit immediately if a command exits with a non-zero exit code.

* Ensure that any copying, installing or compiling delivers resulting code to the install target directory (represented as the 1 environment variable). For `make` commands, use `configure` or its equivalent to accomplish this.

* Be aware that BOSH ensures that dependencies cited in the `dependencies` block of package `spec` files are available to the deployed binary. For example, in the `spec` file for the gogs package, we cite golang1.6 as a dependency. This ensures that on the compilation VMs, the packaging script for gogs has access to the compiled golang1.6 package.

---

# Lab - Create packaging scripts

## golang1.6 packaging script

```
set -e

tar xzf golang/go1.6.3.linux-amd64.tar.gz
cp -R go/* ${BOSH_INSTALL_TARGET}
```

## gogs packaging script

```
set -e

REPO_NAME=github.com/gogits/gogs

export GOROOT=$(readlink -nf /var/vcap/packages/golang1.6)
export GOPATH=$BOSH_INSTALL_TARGET
export PATH=$GOROOT/bin:$PATH

mkdir ${BOSH_INSTALL_TARGET}/src
cp -a * ${BOSH_INSTALL_TARGET}/src

go build \
  -o ${BOSH_INSTALL_TARGET}/bin/gogs \
  -tags "sqlite cert" \
  ${REPO_NAME}

cp -a ${BOSH_INSTALL_TARGET}/src/github.com/gogits/gogs/* ${BOSH_INSTALL_TARGET}/bin
rm -rf ${BOSH_INSTALL_TARGET}/src
```

---

# Lab - Update job specs with dependencies

The dependency graph reveals runtime dependencies that need to be added to the `packages` block of the job spec.

Edit the `gogs_web` job specs to include these dependencies.

In our example, the dependency graph shows that `gogs_web` job depends on `gogs`:

```
packages:
  - gogs
```

---

# Add Blobs

When creating a release, you will likely use a source code repository. But releases often use tar files or other binaries, also known as blobs. Checking blobs into a repository is problematic if your repository unsuited to dealing with large binaries (as is true of Git, for example).

BOSH lets you avoid checking blobs into a repository by doing the following:
* For dev releases, use local copies of blobs.
* For a final release, upload blobs to a blobstore, and direct BOSH to obtain the blobs from there.

# Lab - Configure a blobstore

In the `config` directory, you record the information BOSH needs about the blobstore:
* The `final.yml` file names the blobstore and declares its type, which is either `local` or one of several other types that specify blobstore providers.
* The `private.yml` file specifies the blobstore path, along with a secret.

`private.yml` contains keys for accessing the blobstore and should not be checked into a repository. (Since we used the `--git` option when running `bosh init release` at the beginning of this lab, `private.yml` is automatically `gitignored`.)

The `config` directory also contains a file whose content is automatically generated: the `blobs.yml` file.

We are going to use the `local` type blobstore.

The `local` type blobstore is suitable for learning but the resulting release cannot be shared.

## config/final.yml

```
---
blobstore:
  provider: local
  options:
    blobstore_path: /home/ubuntu/workspace/gogs_blobstore
final_name: gogs
```

## config/private.yml:

```
---
blobstore_secret: 'does-not-matter'
blobstore:
  local:
    blobstore_path: /home/ubuntu/workspace/gogs_blobstore
```

---

# Lab - Add Blobs

In the package `spec` file, the `files` block lists any binaries you downloaded, along with the URLs from which you downloaded them.

Those files are blobs, and now you need the paths to the downloaded blobs on your local system.

```
cd ~/workspace/gogs-release
bosh add blob ~/workspace/go1.6.3.linux-amd64.tar.gz golang
```

The bosh add blob command adds a local blob to the collection your release recognizes as BOSH blobs.

---

# Lab - Create Job Properties

Your service needs to be configurable at deployment time, you create the desired inputs or controls and specify them in the release. Each input is a _property_ that belongs to a particular job.

Creating properties requires three steps:

1. Define properties and defaults in the properties block of the job spec.

2. Use the property lookup helper p() to reference properties in relevant templates.

3. Specify the property in the `deployment manifest`.

PLACEHOLDER

---

# Lab - Create a Dev Release

All the elements needed to create a dev release should now be in place.

For the dev release, use the `--force` option with the `bosh create release` command. This forces BOSH to use the local copies of our blobs.

Without the `--force` option, BOSH requires blobs to be uploaded before you run `bosh create release`. For a final release, we upload blobs, but not for a dev release.

```
cd ~/workspace/gogs-release
bosh create release --force
```

BOSH prompts for a release name, and assigns a dot-number version to the release.

## Upload the new dev release.

```
cd ~/workspace/gogs-release
bosh upload release
```

---

# Lab - Create Release Manifest

# gogs.yml

```
---
name: gogs

# replace with `bosh status --uuid`
director_uuid: REPLACE_ME

# first need to upload the gogs releases
releases:
- name: gogs
  version: latest

stemcells:
- alias: trusty
  os: ubuntu-trusty
  version: latest

instance_groups:
- name: gogs
  instances: 1
  vm_type: default
  stemcell: trusty
  azs: [z1]
  networks: [{name: default}]
  jobs:
  - name: gogs_web
    release: gogs

update:
  canaries: 1
  max_in_flight: 1
  serial: false
  canary_watch_time: 1000-60000
  update_watch_time: 1000-60000
```
