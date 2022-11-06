# Rekor

[Rekor](https://docs.sigstore.dev/rekor/overview) is Sigstore's signature transparency log.
Rekor requires to deploy the [Trillian](https://github.com/google/trillian) log signer and log server and a database to store the signature data. In this example, we will use MariaDB.

## Log into the Rekor instance with SSH

Log into Rekor using the public DNS name set up in the previous section:
```
ssh -i path/to/MyKeyPair.pem instance-username@rekor.sigstore-aws-example.com
```
The `instance-username` for Ubuntu instances is usually `ubuntu`. If you want to ensure this is the correct default username, read the AMI usage instructions.

## Dependencies

Let's install Rekor dependencies.
Start by updating your system:
```
sudo apt-get update -y
```

Remove `man-db`:
```
sudo apt-get remove -y man-db
```

Install the following packages:
```
sudo apt-get install mariadb-server git redis-server haproxy certbot -y
```

> 📝 redis-server is optional, but useful for a quick indexed search should you decide you need it. If you don't install it, you need to start rekor with `--enable_retrieve_api=false`.

## Install latest Golang compiler

Download and run the golang installer:
```
curl -O https://storage.googleapis.com/golang/getgo/installer_linux
```
```
chmod +x installer_linux
```
```
./installer_linux
```

You should see a similar output:
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

Run:
```
source /home/ubuntu/.bash_profile
```

Verify the Go version:
```
go version
    go version go1.19.3 linux/amd64
```

## Install Rekor

Install the Rekor repository:

```
mkdir -p ~/go/src/github.com/sigstore && cd "$_"
```
```
git clone https://github.com/sigstore/rekor.git && cd rekor/
```

Install the server and the CLI:
```
go build -o rekor-cli ./cmd/rekor-cli
```
```
sudo mv rekor-cli /usr/local/bin/
```
```
go build -o rekor-server ./cmd/rekor-server
```
```
sudo mv rekor-server /usr/local/bin/
```

## Set up the database backend for Trillian

Run `mysql_secure_installation` to remove test accounts:

```
sudo mysql_secure_installation
```

```
NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user. If you've just installed MariaDB, and
haven't set the root password yet, you should just press enter here.

Enter current password for root (enter for none): 
OK, successfully used password, moving on...

Setting the root password or using the unix_socket ensures that nobody
can log into the MariaDB root user without the proper authorisation.

You already have your root account protected, so you can safely answer 'n'.

Switch to unix_socket authentication [Y/n] n
 ... skipping.

You already have your root account protected, so you can safely answer 'n'.

Change the root password? [Y/n] n
 ... skipping.

By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] Y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] Y
 ... Success!

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] Y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] Y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
```

Let's build the database.
Edit the `scripts/createdb.sh` within the `rekor` repository and eventually populate the `ROOTPASS`.

```
cd scripts
```
```
sudo ./createdb.sh
```

```
Creating test database and test user account
Loading table data..
```

## Install Trillian components

```
go install github.com/google/trillian/cmd/trillian_log_server@v1.3.14-0.20210713114448-df474653733c
```
```
sudo mv ~/go/bin/trillian_log_server /usr/local/bin/
```
```
go install github.com/google/trillian/cmd/trillian_log_signer@v1.3.14-0.20210713114448-df474653733c
```
```
sudo mv ~/go/bin/trillian_log_signer /usr/local/bin/
```

>  📝 The repository version pinned here correspond to the ones in the original GCP Sigstore the Hard Way tutorial to ensure components compatibility. Those version can eventually be updated to a later version.

## Run Trillian

Run those commands in separate terminals (for debugging purposes):

```
trillian_log_server -http_endpoint=localhost:8090 -rpc_endpoint=localhost:8091 --logtostderr ...
```
```
trillian_log_signer --logtostderr --force_master --http_endpoint=localhost:8190 -rpc_endpoint=localhost:8191  --batch_size=1000 --sequencer_guard_window=0 --sequencer_interval=200ms
```

Alternatively, create bare minimal systemd services:
```
cat /etc/systemd/system/trillian_log_server.service
```
```
[Unit]
Description=trillian_log_server
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=600
StartLimitBurst=5

[Service]
ExecStart=/usr/local/bin/trillian_log_server -http_endpoint=localhost:8090 -rpc_endpoint=localhost:8091 --logtostderr ...
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

```
cat /etc/systemd/system/trillian_log_signer.service
```
```
[Unit]
Description=trillian_log_signer
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=600
StartLimitBurst=5

[Service]
ExecStart=/usr/local/bin/trillian_log_signer --logtostderr --force_master --http_endpoint=localhost:8190 -rpc_endpoint=localhost:8191  --batch_size=1000 --sequencer_guard_window=0 --sequencer_interval=200ms
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Enable systemd services:
```
sudo systemctl daemon-reload
```

```
sudo systemctl enable trillian_log_server.service
```
```
Created symlink /etc/systemd/system/multi-user.target.wants/trillian_log_server.service → /etc/systemd/system/trillian_log_server.service.
sudo systemctl start trillian_log_server.service
sudo systemctl status trillian_log_server.service
● trillian_log_server.service - trillian_log_server
   Loaded: loaded (/etc/systemd/system/trillian_log_server.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2021-09-30 17:41:49 UTC; 8s ago
```

```
sudo systemctl enable trillian_log_signer.service
```
```
Created symlink /etc/systemd/system/multi-user.target.wants/trillian_log_signer.service → /etc/systemd/system/trillian_log_signer.service.
sudo systemctl start trillian_log_signer.service
sudo systemctl status trillian_log_signer.service
● trillian_log_signer.service - trillian_log_signer
   Loaded: loaded (/etc/systemd/system/trillian_log_signer.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2021-09-30 17:42:05 UTC; 12s ago
```

## Start Rekor

```
rekor-server serve --rekor_server.address=0.0.0.0 --trillian_log_server.port=8091
```

> 📝 Rekor runs on port 3000 on all interfaces by default

Alternatively, you may create a bare minimal systemd service similar to trillian above:

```
cat /etc/systemd/system/rekor.service
```
```
[Unit]
Description=rekor
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=600
StartLimitBurst=5

[Service]
ExecStart=/usr/local/bin/rekor-server serve --rekor_server.address=0.0.0.0 --trillian_log_server.port=8091
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl daemon-reload
```
```
sudo systemctl enable rekor.service
```
```
sudo systemctl start rekor.service
```
```
sudo systemctl status rekor.service
```

## Let's encrypt (TLS) & HA Proxy config

Let's create a HAProxy config, set `DOMAIN` to your registered domain and your private IP address.

```
DOMAIN="rekor.example.com"
IP="172.31.19.164"
```

Let's now run certbot to obtain our TLS certs.

```
sudo certbot certonly --standalone --preferred-challenges http \
      --http-01-address ${IP} --http-01-port 80 -d ${DOMAIN} \
      --non-interactive --agree-tos --email youremail@domain.com
```

