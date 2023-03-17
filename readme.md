# Install and Configure etcd Cluster with TLS

This doc walks though installing and configuring a 'semi-secure' etcd cluster. It uses the same self-signed cert and key for all requirements. It is intended for lab use only and is not production grade. This follows from my blog post at https://vrelevant.net/install-etcd-cluster-with-tls/

This guide uses the 'static' cluster configuration method. For more info on clustering options, review the official docs: https://etcd.io/docs/v3.5/op-guide/clustering/.

It also installs etcd on a base Ubuntu operating system. It could just as well be done with a container image and run on a container host.

## Create Required Certificates

With three Ubuntu nodes provisioned, from your first node, create a temporary working directory and perform the following commands from that context. At the end of this section, you'll have all of the certs setup on each node and be ready to install etcd on each node.

The etcd.io docs use cfssl bin to create the certificates. I didn't see any glaring reason to download and learn a new tool to do the same thing an existing tool does. So, I used openssl.

1. Create openssl config file (Change the IP addresses to match your needs, but leave
   127.0.0.1):

```console
cat <<EOF > etcd_cert.cnf
[req]
default_bits  = 4096
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
countryName = US
stateOrProvinceName = MI
localityName = Detroit
organizationName = Lab
commonName = etcd-host

[v3_req]
keyUsage = digitalSignature, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names

[alt_names]
IP.1 = 127.0.0.1
IP.2 = 192.168.140.1
IP.3 = 192.168.140.2
IP.4 = 192.168.140.3
DNS.1 = localhost
EOF
```

2. Create CA self-signed cert

```bash
openssl req -x509 -noenc -newkey rsa:4096 -subj '/CN=labCA' -keyout ca_key.pem -out ca_cert.pem -days 36500
```

3. Create key and certificate signing request for etcd TLS

```bash
openssl req -noenc -newkey rsa:4096 -keyout etcd_key.pem -out etcd_cert.csr -config etcd_cert.cnf
```

4. Verify the config file create CSR with the subject alternate names and extensions request

```bash
openssl req -in etcd_cert.csr  -noout -text
```

5. Create certificate

```bash
openssl x509 -req -days 36500 -in etcd_cert.csr -CA ca_cert.pem -CAkey ca_key.pem -out etcd_cert.pem -copy_extensions copy
```

6. Verify subject alternate names and extensions were copied to certificate

```bash
openssl x509 -text -noout -in etcd_cert.pem
 ```

7. Create directories for etcd certs

```bash
sudo mkdir -p /etc/ssl/etcd/certificate
sudo mkdir /etc/ssl/etcd/private
```

8. Copy ca_cert.pem and etcd_cert.pem into the certificate directory. Copy etcd_key.pem into the private directory

```bash
sudo cp ca_cert.pem etcd_cert.pem /etc/ssl/etcd/certificate
sudo cp etcd_key.pem /etc/ssl/etcd/private
```

9. Create the same directories on your other two nodes, and SCP the certs and key to them


## Install and Configure etcd

Perform the following on each node:

1. Download latest etcd bins (Check for latest release at https://github.com/etcd-io/etcd/releases)

```bash
export ETCD_RELEASE="v3.5.7"
```

```bash
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_RELEASE}/etcd-${ETCD_RELEASE}-linux-amd64.tar.gz | tar xvz
```

```bash
cd etcd-${ETCD_RELEASE}-linux-amd64
```

2. Copy etcd bins to /usr/bin

```bash
sudo cp -p etcd etcdctl etcdutl /usr/bin
```

3. Create directory for etcd data

```bash
sudo mkdir -p /var/lib/etcd/
```

4. Create user account for etcd service and assign directory permissions

```bash
sudo groupadd -r etcd
```

```bash
sudo useradd -c "etcd service" --shell /usr/sbin/nologin -r etcd -g etcd
```

```bash
sudo chown -R etcd:etcd /var/lib/etcd && sudo chown -R etcd:etcd /etc/ssl/etcd
```

```bash
sudo chmod -R 700 /var/lib/etcd && sudo chmod -R 550 /etc/ssl/etcd
```

All files are now where they need to be, you can delete the directory you've been using to create certificates and download etcd files to.

5. Create etcd systemd unit file

```console
cat <<EOF | sudo tee -a /etc/systemd/system/etcd.service
[Unit]
Description=etcd service
Documentation=https://etcd.io/docs
Documentation=man:etcd
After=network.target
Wants=network-online.target

[Service]
Environment=ETCD_NAME=%H
Environment=ETCD_DATA_DIR=/var/lib/etcd
EnvironmentFile=-/etc/default/etcd
Type=notify
User=etcd
PermissionsStartOnly=true
ExecStart=/usr/bin/etcd $DAEMON_ARGS
Restart=on-abnormal
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

6. Create etcd service config (Change node names and IPs for cluster in first line to match your needs)

```bash
export ETCD_HOST_IP=<current node IP>
```

```console
cat <<EOF | sudo tee -a /etc/default/etcd

## Set <node>=<url> for each occurence to match the nodes in your cluster
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.140.1:2380,etcd-2=https://192.168.140.2:2380,etcd-3=https://192.168.140.3:2380"

ETCD_NAME="${HOSTNAME}"
ETCD_LISTEN_PEER_URLS="https://${ETCD_HOST_IP}:2380"
ETCD_LISTEN_CLIENT_URLS="https://127.0.0.1:2379,https://${ETCD_HOST_IP}:2379"
ETCD_ADVERTISE_CLIENT_URLS="https://${ETCD_HOST_IP}:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://${ETCD_HOST_IP}:2380"

##Leave the rest of this file as-is
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-1"
ETCD_CERT_FILE="/etc/ssl/etcd/certificate/etcd_cert.pem"
ETCD_KEY_FILE="/etc/ssl/etcd/private/etcd_key.pem"
ETCD_PEER_TRUSTED_CA_FILE="/etc/ssl/etcd/certificate/ca_cert.pem"
ETCD_PEER_CERT_FILE="/etc/ssl/etcd/certificate/etcd_cert.pem"
ETCD_PEER_KEY_FILE="/etc/ssl/etcd/private/etcd_key.pem"
ETCD_PEER_CLIENT_CERT_AUTH=true
EOF
```
7. Repeat this section on remaining nodes

## Enable and Start etcd

Once all of the nodes have etcd installed and configured, we can enable and start the service on each. They will peer with each other and elect a leader.

1. Enable and start etcd service

On each node:

```bash
sudo systemctl daemon-reload && sudo systemctl enable etcd
```

```bash
sudo systemctl start etcd
```

## Check for Success

1. From an etcd node, set env vars for etcdctl

```bash
export ETCDCTL_CACERT=/etc/ssl/etcd/certificate/ca_cert.pem
export ETCDCTL_CERT=/etc/ssl/etcd/certificate/etcd_cert.pem
export ETCDCTL_KEY=/etc/ssl/etcd/private/etcd_key.pem
```

2. Add your user to the etcd group to provide access to the TLS certs

```bash 
sudo usermod -a -G etcd <user name>
```

```bash
su <user name>
```

3. Check etcd cluster member list (Change IP to match your needs)

```bash
etcdctl --endpoints=https://192.168.140.1:2379,https://192.168.140.2:2379,https://192.168.140.3:2379 endpoint status -w table
```
![image](https://user-images.githubusercontent.com/45366367/225124098-85fe1468-ea18-42c9-881b-347a28ef2d82.png)

4. Create and retrieve a key/value

```bash
etcdctl --endpoints=https://192.168.140.1:2379 put greeting "Hello, vRelevant"
```

```bash
etcdctl --endpoints=https://192.168.140.1:2379 get greeting

etcdctl --endpoints=https://192.168.140.2:2379 get greeting

etcdctl --endpoints=https://192.168.140.3:2379 get greeting
```

#### That's it! If all went well, you now have a three node etcd cluster. You can use this cluster for anything your key/value pair heart desires.
