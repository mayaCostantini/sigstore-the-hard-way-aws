# Dex

Dex is the solution used for handling OpenID connect sessions.

A user first connects to a dex instance where an OpenID session is invoked.

The user then authorises Fulcio to request the users email address as part of an OpenID scope. This email address is then recored into the x509 signing certificates

Connect to the `oauth2` compute instance
```
ssh -i MyKeyPair.pem ubuntu@oauth2.sigstore-aws-example.com
```

## Dependencies

```
sudo apt-get update -y
```

If you want to save up some time, remove man-db first

```
sudo apt-get remove -y --purge man-db
```
```
sudo apt-get install haproxy make git gcc certbot -y
```

### Install latest golang compiler

Download and run the golang installer (system package is not yet 1.16)

```
curl -O https://storage.googleapis.com/golang/getgo/installer_linux
```
```
chmod +x installer_linux
```
```
./installer_linux
```

e.g.

```
Welcome to the Go installer!
Downloading Go version go1.19.3 to /home/ubuntu/.go
This may take a bit of time...
Downloaded!
Setting up GOPATH
GOPATH has been set up!

One more thing! Run `source /home/ubuntu/.bash_profile` to persist the
new environment variables to your current session, or open a
new shell prompt.
```

As suggested run

```
source /home/$USER/.bash_profile
```
```
go version
```
```
go version go1.19.3 linux/amd64
```

## Let's encrypt (TLS) & HA Proxy config

Let's create a HAProxy config, set `DOMAIN` to your registered domain and your private `IP` address
```
DOMAIN="oauth2.sigstore-aws-example.com"
IP="172.31.23.76"
```

Let's now run certbot to obtain our TLS certs.
```
sudo certbot certonly --standalone --preferred-challenges http \
      --http-01-address ${IP} --http-01-port 80 -d ${DOMAIN} \
      --non-interactive --agree-tos --email youremail@domain.com
```

Move the PEM chain into place
```
sudo cat "/etc/letsencrypt/live/${DOMAIN}/fullchain.pem" \
    "/etc/letsencrypt/live/${DOMAIN}/privkey.pem" \
    | sudo tee "/etc/ssl/private/${DOMAIN}.pem" > /dev/null
```

Now we need to change certbot configuration for automatic renewal
Prepare post renewal script
```
sudo vim /etc/letsencrypt/renewal-hooks/post/haproxy-ssl-renew.sh
```
```
#!/bin/bash

DOMAIN="oauth2.example.com"

cat "/etc/letsencrypt/live/${DOMAIN}/fullchain.pem" \
    "/etc/letsencrypt/live/${DOMAIN}/privkey.pem" \
    > "/etc/ssl/private/${DOMAIN}.pem"

systemctl reload haproxy.service
```

Make sure the script has executable flag set
```
sudo chmod +x /etc/letsencrypt/renewal-hooks/post/haproxy-ssl-renew.sh
```

Replace port and address in the certbot's renewal configuration file for the domain (pass ACME request through the haproxy to certbot)
```
sudo vim /etc/letsencrypt/renewal/oauth2.example.com.conf
```
```
http01_port = 9080
http01_address = 127.0.0.1
```

Append new line
```
post_hook = /etc/letsencrypt/renewal-hooks/post/haproxy-ssl-renew.sh
```

Prepare haproxy configuration
```
cat > haproxy.cfg <<EOF
defaults
    timeout connect 10s
    timeout client 30s
    timeout server 30s
    log global
    mode http
    option httplog
    maxconn 3000
    log 127.0.0.1 local0

frontend haproxy
    #public IP address
    bind ${IP}:80
    bind ${IP}:443 ssl crt /etc/ssl/private/${DOMAIN}.pem

    # HTTPS redirect
    redirect scheme https code 301 if !{ ssl_fc }

    acl letsencrypt-acl path_beg /.well-known/acme-challenge/
    use_backend letsencrypt-backend if letsencrypt-acl

    default_backend sigstore_dex

backend sigstore_dex
    server sigstore_oauth2_internal ${IP}:6000

backend letsencrypt-backend
    server certbot_internal 127.0.0.1:9080
EOF
```

Inspect the resulting `haproxy.cfg` and make sure everything looks correct.
If so, move it into place
```
sudo mv haproxy.cfg /etc/haproxy/
```

Check syntax
```
sudo /usr/sbin/haproxy -c -V -f /etc/haproxy/haproxy.cfg
```

### Start HAProxy

Let's now start HAProxy
```
sudo systemctl enable haproxy.service
```
```
Synchronizing state of haproxy.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable haproxy
```

```
sudo systemctl restart haproxy.service
```
```
sudo systemctl status haproxy.service
```

Test automatic renewal
```
sudo certbot renew --dry-run
```

### Install Dex

```
mkdir -p ~/go/src/github.com/dexidp/ && cd "$_"
git clone https://github.com/dexidp/dex.git
```
```
cd dex
make build
sudo mv bin/dex /usr/local/bin/
```

### Obtain Google OAUTH credentials

> 📝 We re using Google here, you can do the same for github and microsoft too.
  The placeholders are already within `config.yaml`

1. Head to the [credentials page](https://console.cloud.google.com/apis/credentials)

2. Select 'CONFIGURE CONSENT SCREEN'

    Select 'Internal'

    > If you're not a Google Workspace user, the 'Internal' option will not be available. You can only make your app available to external (general audience) users only. In such a case, the 'External' User Type works fine as well.

    Fill out the app registration details: 

    - App name: sthw-aws
    - User support email: your@email.com
    - Application home page: https://oauth2.sigstore-aws-example.com
    - Authorized domains: sigstore-aws-example.com
    - Developer contact information: your@email.com

3. Set scopes

    Select 'ADD OR REMOVE SCOPES' and set the `userinfo.email` scope

    Select "SAVE AND CONTINUE"

    Select "BACK TO DASHBOARD" and select 'Credentials'

4. Create OAuth Client ID

    Click on "Create credentials" and select "OAuth client ID". Select "Web Application" and fill out the "Authorized Redirect URIs"

    Select "CREATE"

    - Application type: Web application
    - Name: sthw-aws
    - Authorized redirect URIs: https://oauth2.sigstore-aws-example.com/auth

5. Note down tour Client ID and Secret and keep them safe (we will need them for dex)


### Configure Dex

Set up the configuration file for dex.
Provide saved OIDC details as variables
```
GOOGLE_CLIENT_ID="..."
GOOGLE_CLIENT_SECRET="..."
```
```
cat > dex-config.yaml <<EOF
issuer: https://${DOMAIN}/auth

storage:
  type: sqlite3
  config:
    file: /var/dex/dex.db
web:
  http: 0.0.0.0:5556
frontend:
  issuer: sigstore
  theme: light

# Configuration for telemetry
telemetry:
  http: 0.0.0.0:5558

# Options for controlling the logger.
logger:
  level: "debug"
  format: "json"

# Default values shown below
oauth2:
  responseTypes: [ "code" ]
  skipApprovalScreen: false
  alwaysShowLoginScreen: true

staticClients:
  - id: sigstore
    public: true
    name: 'sigstore'
redirectURI: https://${DOMAIN}/auth/callback

connectors:
- type: google
  id: google-sigstore-test
  name: Google
  config:
    clientID: $GOOGLE_CLIENT_ID
    clientSecret: $GOOGLE_CLIENT_SECRET
    redirectURI: https://${DOMAIN}/auth/callback

#- type: microsoft
#  id: microsoft-sigstore-test
#  name: Microsoft
#  config:
#     clientID: $MSFT_CLIENT_ID
#     clientSecret: $MSFT_CLIENT_SECRET
#     redirectURI: https://${DOMAIN}/auth/callback

#- type: github
#  id: github-sigstore-test
#  name: GitHub
#  config:
#     clientID: $GITHUB_CLIENT_ID
#     clientSecret: $GITHUB_CLIENT_SECRET
#     redirectURI: https://${DOMAIN}/auth/callback
EOF
```

>  SQLite3 is the recommended storage for users who want to stand up dex quickly. It is not appropriate for real workloads (see [Dex storage documentation](https://dexidp.io/docs/storage/#sqlite3)).

Move configuration file
```
sudo mkdir -p /var/dex/
sudo mkdir -p /etc/dex/
sudo mv dex-config.yaml /etc/dex/
```

### Start dex

Give the appropriate database file permissions for the server
```
sudo chmod -R 660 /var/dex/dex.db
```

Start the server
```
sudo dex serve --web-http-addr=0.0.0.0:6000  /etc/dex/dex-config.yaml
```
