# Certificate transparency log

We will now install the Certificate transparency log (CTL).

CTL requires running instances of trillian's log server and signer

Let's start by logging in (specify the public IP address associated with the `sigstore-ctl` instance)
```
ssh -i MyKeyPair.pem ubuntu@34.254.196.47
```

## Dependencies

```bash
sudo apt-get update -y
```

If you want to save up some time, remove man-db first

```bash
sudo apt-get remove -y --purge man-db
```

```bash
sudo apt-get install mariadb-server git wget -y
```

### Install latest golang compiler

Download and run the golang installer (system package is not yet 1.16)

```bash
curl -O https://storage.googleapis.com/golang/getgo/installer_linux
```

```bash
chmod +x installer_linux
```

```bash
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

```bash
source /home/$USER/.bash_profile
go version
go version go1.19.3 linux/amd64
```

### Database

Trillian requires a databbase, let's first run `mysql_secure_installation`

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

We can now import the database as we used for rekor
```
wget https://raw.githubusercontent.com/sigstore/rekor/main/scripts/createdb.sh
```
```
wget https://raw.githubusercontent.com/sigstore/rekor/main/scripts/storage.sql
```
```
chmod +x createdb.sh
```
```
sudo ./createdb.sh
```

E.g.
```
Creating test database and test user account
Loading table data..
```

### Install trillian components

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
```
go install github.com/google/trillian/cmd/createtree@v1.3.14-0.20210713114448-df474653733c
```
```
sudo mv ~/go/bin/createtree /usr/local/bin/
```

### Run trillian

The following is best run in two terminals which are then left open (this helps for debugging)
```
trillian_log_server -http_endpoint=localhost:8090 -rpc_endpoint=localhost:8091 --logtostderr ...
```
```
trillian_log_signer --logtostderr --force_master --http_endpoint=localhost:8190 -rpc_endpoint=localhost:8191  --batch_size=1000 --sequencer_guard_window=0 --sequencer_interval=200ms
```

Alternatively, create bare minimal systemd services

```bash
cat /etc/systemd/system/trillian_log_server.service
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

```bash
cat /etc/systemd/system/trillian_log_signer.service
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

Enable systemd services

```bash
sudo systemctl daemon-reload
```

```bash
sudo systemctl enable trillian_log_server.service
Created symlink /etc/systemd/system/multi-user.target.wants/trillian_log_server.service → /etc/systemd/system/trillian_log_server.service.
sudo systemctl start trillian_log_server.service
sudo systemctl status trillian_log_server.service
● trillian_log_server.service - trillian_log_server
   Loaded: loaded (/etc/systemd/system/trillian_log_server.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2021-09-30 17:41:49 UTC; 8s ago
```

```bash
sudo systemctl enable trillian_log_signer.service
Created symlink /etc/systemd/system/multi-user.target.wants/trillian_log_signer.service → /etc/systemd/system/trillian_log_signer.service.
sudo systemctl start trillian_log_signer.service
sudo systemctl status trillian_log_signer.service
● trillian_log_signer.service - trillian_log_signer
   Loaded: loaded (/etc/systemd/system/trillian_log_signer.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2021-09-30 17:42:05 UTC; 12s ago
```

### Install CTFE server

```bash
go install github.com/google/certificate-transparency-go/trillian/ctfe/ct_server@latest
```

```bash
sudo mv ~/go/bin/ct_server /usr/local/bin/
```

### Create a private key

> **Warning**
> The following section dumps out keys into the home directory. This is only recommended if you do not greatly care about the security of this machine
> If you do care, place them into a more secure location and chmod to a secure level of file permissions.

Create a key pair with the following command:

```bash
openssl ecparam -genkey -name prime256v1 -noout -out unenc.key
openssl ec -in unenc.key -out privkey.pem -des
```

Extract the public key from the key-pair:

```bash
openssl ec -in privkey.pem -pubout -out ctfe_public.pem
```

Feel free to remove the unencrypted key:

```bash
rm unenc.key
```

> **Note**
> The private key needs a passphrase, remember it as you will need it for `your_passphrase` when we create the `ct.cfg` further down.

> **Note**
> You will need the ctfe_public.pem file for the TUF root of cosign, with the sign-container section towards the end

### Create a Tree ID

> **Note**
> `trillian_log_server` needs to be running for this command to execute

```bash
LOG_ID="$(createtree --admin_server localhost:8091)"
```

### Set up the config file

```bash
cat > ct.cfg <<EOF
config {
  log_id: ${LOG_ID}
  prefix: "sigstore"
  roots_pem_file: "/etc/ctfe-config/fulcio-root.pem"
  private_key: {
    [type.googleapis.com/keyspb.PEMKeyFile] {
       path: "/etc/ctfe-config/privkey.pem"
       password: "your_passphrase"
    }
  }
}
EOF
```

Afterwards, open the file again and change `<your_passphrase>` to the one you used
when generating the private key.

> **Note**
> `fulcio-root.pem` is the root ID certificate, we created in [06-fulcio](06-fulcio.md).

```bash
sudo mkdir -p /etc/ctfe-config/
sudo mv ct.cfg /etc/ctfe-config/
sudo mv fulcio-root.pem /etc/ctfe-config/
sudo mv privkey.pem /etc/ctfe-config/
```