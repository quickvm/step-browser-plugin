# step-browser-plugin

The step-browser-plugin automates the usage of TLS certificates for client authentication in Chrome and Firefox browsers on Linux and Google Chrome.app, Firefox.app, Safari.app on MacOS. It also supports CLI applications.

## Install

### Requirements

* A free [Smallstep Certificate Manager](https://smallstep.com/certificate-manager/pricing/) account (or [step-ca](https://smallstep.com/certificates/index.html))
* [step CLI](https://smallstep.com/cli/)
* [jq](https://stedolan.github.io/jq/)
* [yq](https://mikefarah.gitbook.io/yq/)
* certutil and pk12util: Debian/Ubuntu: `sudo apt install libnss3-tools` Fedora/RHEL: `sudo dnf install nss-tools`
* An overwhelming desire to use certificate client auth in your daily life

### Bootstrap your CA

```bash
step ca bootstrap --ca-url ${SMALLSTEP_CERTIFICATE_MANAGER_CA_URL} --fingerprint ${SMALLSTEP_CERTIFICATE_MANAGER_CA_FINGERPRINT} --context home --install
```

The `step browser` plugin has support for step CLI contexts. This allows you to use many different CAs for different environments. In this example we just bootstraped a CA in the `home` context above and we will select the `home` context with `step context select`.

### Select step CLI context

```bash
step context select home
✔ Context: home
step context list
▶ home
work
```

### Install the step-browser-plugin

Clone this git repo and run `./install` from the git repo root. This will the plugin to `${STEPPATH}/plugins/` directory.

### Create OIDC Provisioner

You will want to create a OIDC Provisioner to use to get a certificate from Certificate Manager. Read more about creating a smallstep OIDC Provisioner on the [documentation site](https://smallstep.com/docs/step-ca/provisioners#oauthoidc-single-sign-on).

```bash
step ca provisioner add Google --type oidc \
  --client-id myclientid-url.apps.googleusercontent.com \
  --client-secret clientsecret \
  --configuration-endpoint https://accounts.google.com/.well-known/openid-configuration \
  --domain smallstep.com --listen-address=127.0.0.1:10000
```

### Run step browser login to configure the plugin for the first time

You will need to use your email address, and the OIDC provisioner name that you created on your Certificate Manager account or step-ca instance.

```bash
step browser login
Config file /home/username/.step/authorities/home/config/config.yml not found for the https://home.yoursmallstep.ca.smallstep.com CA. Configuring...
Enter email address: you@example.com
OIDC Provisioner name: Google
Cert duration? (2h30m20s format -- min: 5m0s max: 24h0m0s)
Duration [enter for default: 8h0m0s]:

Write:

email: you@example.com
provisioner: Google
duration: 8h0m0s
to /home/username/.step/authorities/home/config/config.yml?

Confirm: [y/N] y
/home/username/.step/authorities/home/config/config.yml written!

You are logged in with a certificate from https://home.yoursmallstep.ca.smallstep.com CA!
source /home/username/.step/cert_file_vars
```

This will write out a `/home/username/.step/authorities/home/config/config.yml` configuration file for the plugin and it will add your certificate to Chrome and Firefox profiles on Linux and for MacOS it will add it to Google Chrome.app, Firefox.app, Safari.app. It will also write out a `/home/username/.step/cert_file_vars` file which can be used to source into your shell environment. This will allow you can use your CLI apps with your certificates. By default it only creates these environment variables:

```bash
$ cat /home/username/.step/cert_file_vars
export BROWSER_CA_FILE=/home/username/.step/authorities/home/certs/root_ca.crt
export BROWSER_CRT_FILE=/home/username/.step/authorities/home/certs/username.crt
export BROWSER_KEY_FILE=/home/username/.step/authorities/home/certs/username.key
```

`step browser edit` allows you to create a `.step/authorities/home/config/env_hook` file per `step context` that will allow you to include other environment variables to `/home/username/.step/cert_file_vars`. In this example we are going to add in support for Hashicorp Nomad and Vault. Run `step browser edit` which will run `$EDITOR ${BROWSER_CONFIG_DIR}/env_hook` to edit the `env_hook` file for the current configuration. We are going to add the environment variables below:

```
NOMAD_ADDR=https://192.168.1.101:4646
NOMAD_CACERT="${BROWSER_CA_FILE}"
NOMAD_CLIENT_CERT="${BROWSER_CRT_FILE}"
NOMAD_CLIENT_KEY="${BROWSER_KEY_FILE}"
NOMAD_TLS_SERVER_NAME=localhost
NOMAD_HTTP_SSL=true
VAULT_ADDR=https://192.168.1.101:8200
VAULT_CACERT="${BROWSER_CA_FILE}"
VAULT_CLIENT_CERT="${BROWSER_CRT_FILE}"
VAULT_CLIENT_KEY="${BROWSER_KEY_FILE}"
VAULT_TLS_SERVER_NAME=localhost
```

Save the file and quit the editor. Then run `step browser export` which will recreate `/home/username/.step/cert_file_vars` for the current `step context` that you are using. You can see below the added environment variables are now added to `/home/username/.step/cert_file_vars`

```bash
step browser export
export BROWSER_CA_FILE=/home/username/.step/authorities/home/certs/root_ca.crt
export BROWSER_CRT_FILE=/home/username/.step/authorities/home/certs/username.crt
export BROWSER_KEY_FILE=/home/username/.step/authorities/home/certs/username.key
export NOMAD_ADDR=https://192.168.1.101:4646
export NOMAD_CACERT="${BROWSER_CA_FILE}"
export NOMAD_CLIENT_CERT="${BROWSER_CRT_FILE}"
export NOMAD_CLIENT_KEY="${BROWSER_KEY_FILE}"
export NOMAD_TLS_SERVER_NAME=localhost
export NOMAD_HTTP_SSL=true
export VAULT_ADDR=https://192.168.1.101:8200
export VAULT_CACERT="${BROWSER_CA_FILE}"
export VAULT_CLIENT_CERT="${BROWSER_CRT_FILE}"
export VAULT_CLIENT_KEY="${BROWSER_KEY_FILE}"
export VAULT_TLS_SERVER_NAME=localhost
```

You can then run `source ${HOME}/.step/cert_file_vars` to enable the above environment vars in your current shell. You can also add `source ${HOME}/.step/cert_file_vars` to your shell's rc (.bashrc or .zshrc for example) to auto source them when you create a new shell. You should now be able to use CLI apps with your certificates.

```bash
$ nomad job status
ID           Type     Priority  Status   Submit Date
acme-ra      service  50        running  2023-05-26T01:05:37-05:00
coredns      system   50        running  2023-05-26T01:34:10-05:00
traefik      service  50        running  2023-05-26T01:48:00-05:00
whoami       service  50        running  2023-05-23T00:26:18-05:0
```

## Licenses

MIT License

Copyright (c) 2023 QuickVM

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
