# Platform

## EPR - Elastic Package Registry

### Overview

Elastic Package Registry referred as EPR, is a service designed to streamline the management and distribution of integrations packages. EPR offers both public and private repositories for package storage and hosting, providing users with a secure, reliable way to share their custom integrations and plugins with others.

EPR simplifies the process by allowing users to create, manage, and publish their own packages. These packages can be easily installed and updated within Kibana using the WebUI. This not only saves time for developers and users but also ensures consistency in deployments across various environments.

### EPR Architecture

```mermaid
flowchart TD
    A[Kibana 1] -- HTTP(S) --> B[Elastic Package Registry]
    C[Kibana 2] -- HTTP(S)--> B
    D[Kibana 3] -- HTTP(S)--> B
    E[Kibana 4] -- HTTP(S)--> B
    B --> packages@{ shape: disk }
```

### EPR Installation

#### EPR Binary

**Golang**

Remove any previous Go installation by deleting the /usr/local/go folder (if it exists), then extract the archive you just downloaded into /usr/local, creating a fresh Go tree in /usr/local/go:

```bash
curl -O https://dl.google.com/go/go1.23.1.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.23.1.linux-amd64.tar.gz
```

(You may need to run the command as root or through sudo).

Do not untar the archive into an existing /usr/local/go tree. This is known to produce broken Go installations.
Add /usr/local/go/bin to the PATH environment variable.

You can do this by adding the following line to your $HOME/.profile or /etc/profile (for a system-wide installation):

```bash
export PATH=$PATH:/usr/local/go/bin
```

Note: Changes made to a profile file may not apply until the next time you log into your computer. To apply the changes immediately, just run the shell commands directly or execute them from the profile using a command such as source $HOME/.profile.
Verify that you've installed Go by opening a command prompt and typing the following command:

```bash
go version
```

Confirm that the command prints the installed version of Go.

**Mage**

Now that you have golang you can install Mage and its dependencies

```bash
git clone https://github.com/magefile/mage
cd mage
go run bootstrap.go
```

Validate it's installed by running the following command

```bash
mage --version
Mage Build Tool v1.15.0-5-g2385abb
Build Date: 2024-09-18T12:24:14Z
Commit: 2385abb
built with: go1.23.1
```

**Elastic Package Registry**

Now that you have all the tools installed you can clone the elastic package registry

```bash
git clone git@github.com:elastic/package-registry.git
cd package-registry
mage build
```

Binary should be in the bin folder, we are now installing onto the system by running the following commands as root user

```bash
mkdir -p /etc/package-registry /var/package-registry
mv package-registry /etc/package-registry/
```

You can validate it's installed by running the following command:

```bash
/etc/package-registry/package-registry -version
Elastic Package Registry version 1.25.1
```

Now that everything is in place, we can configure package-registry through its configuration _config.yml_ file that you can put in the _/etc/package-registry_ folder.

Below an example of the _config.yml_ file:

```bash
# If users want to add their own packages, they should be put under
# /packages/package-registry or the config must be adjusted.
package_paths:
  - /var/package-registry/packages

cache_time.index: 10s
cache_time.search: 10s
cache_time.categories: 10s
cache_time.catch_all: 10s
```

where:

* _package_paths_ : folder(s) where the packages are stored
* cache_time.index: Sets the caching time for the index endpoint (providing registry info). It allows quick updates when a new version is released, suggested at 10 seconds for fast response to registry changes.
* cache_time.search: Caches the /search endpoint, which should update each time a new package version is released. Recommended at 1 minute to limit delay in package visibility.
* cache_time.categories: Defines cache duration for category counters, which are less critical and suggested at 10 minutes. This impacts only counter accuracy during cache time.
* cache_time.catch_all: Caches all other static assets indefinitely but suggested at 1 hour to reduce CDN traffic while allowing periodic updates.

[Reference](https://github.com/elastic/package-storage/issues/390)

To ease the managmeent of the configuration, we can use a systemd unit _/etc/systemd/system/package-registry_ file to start the service automatically on boot :

Simply paste the following content and adapt it if needed:

```bash
[Unit]
Description=Elastic Package Registry Services

[Service]
User=root
WorkingDirectory=/etc/package-registry
ExecStart=/etc/package-registry/package-registry -address 0.0.0.0:8080 -config /etc/package-registry/config.yml 
# optional items below
Restart=always
RestartSec=3

StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=package-registry

[Install]
WantedBy=multi-user.target

```

Then, reload the systemd daemon and enable the service :

```bash
 systemctl daemon-reload
 systemctl enable package-registry --now
```

Your service will be available on the port 8080 of your server. You can check it with a _curl_ command to see if it is working.

```bash
curl http://localhost:8080/search\?package\={PACKAGE_NAME}\&prerelease\=true
```

##### EPR Docker

You can also use the prebuilt docker image :

```bash
docker run --rm -it -p 8080:8080 \
  -v /path/to/local/packages:/packages/package-registry \
  $(docker images -q docker.elastic.co/package-registry/package-registry:main)
```

## Elastic Package Management Script

## Purpose

This script automates the process of managing Elastic packages by:

- Searching for a package in the Elastic Package Registry (EPR) based on the package name and Kibana version.
- Downloading the package from the registry.
- Uploading the package to a specified Kibana instance using the Fleet API.

The script simplifies the workflow for administrators and developers who need to manage Elastic integrations efficiently.

## Usage

### 1. Set Required Environment Variables

Before running the script, ensure the following environment variables are set:

```bash
export KIBANA_URL="https://your-kibana-instance:5601"
export KIBANA_API_KEY="your-kibana-api-key"
```

### 2. Run the Script

```bash
./manage_package.sh <kibana_version> <package_name> [--debug] [--insecure]
```

explanations :

- <kibana_version>: Specify the version of Kibana (e.g., 8.15.2).
- <package_name>: Name of the package to search and upload (e.g., netskope).
- --debug: Enable verbose output for debugging the search and upload process.
- --insecure: Allow insecure connections to the Kibana server (useful for self-signed certificates).

### 3. Examples

```bash
./manage_package.sh 8.15.2 netskope --debug
```

with insecure connection:

```bash
./manage_package.sh 8.15.2 netskope --insecure
```