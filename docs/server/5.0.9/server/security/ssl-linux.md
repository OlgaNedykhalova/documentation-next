# Setting up SSL on Linux

::: tip
This guide uses the latest Ubuntu LTS (18.04)
:::

First, create a private key and self-signed certificate request (only for testing purposes):

```bash
openssl req \
  -x509 -sha256 -nodes -days 365 -subj "/CN=eventstore.com" \
  -newkey rsa:2048 -keyout eventstore.pem -out eventstore.csr
```

Export the p12 file from the certificate request. You use this when starting EventStoreDB:

```bash
openssl pkcs12 -export -inkey eventstore.pem -in eventstore.csr -out eventstore.p12
```

You need to add the certificate to Ubuntu's trusted certificates. Copy the cert to the _ca-certificates_ folder and update the certificates:

```bash
sudo cp eventstore.csr /usr/local/share/ca-certificates/eventstore.crt

sudo update-ca-certificates
```

The mono framework has its own separate certificate store which you need to sync with the changes you made to Ubuntu's certificates.

You first need to install `mono-devel` version 5.16.0 :

```bash
sudo apt install gnupg ca-certificates
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF
# Update "bionic" to match your Ubuntu version
echo "deb https://download.mono-project.com/repo/ubuntu stable-bionic/snapshots/5.16.0 main" | sudo tee /etc/apt/sources.list.d/mono-official-stable.list
sudo apt update

sudo apt-get install mono-devel
```

This process installs `cert-sync`, which you use to update mono's certificate store with the new certificate:

```bash
sudo cert-sync eventstore.csr
```

Start EventStoreDB node with the following configuration in the configuration file:

```yaml
CertificateFile: eventstore.p12
ExtSecureTcpPort: 1115
```

Connect to Event Store:

::::: tabs
:::: tab .NET API

```csharp
var settings = ConnectionSettings
    .Create()
    .UseSslConnection("eventstore.com", true);

using var conn = EventStoreConnection
    .Create(settings, new IPEndPoint(IPAddress.Loopback, 1115));

await conn.ConnectAsync();
```

::::
:::: tab HTTP API

```bash
curl -vk --cert <PATH_TO_CERT> --key <PATH_TO_KEY> -i -d "@event.json" "http://127.0.0.1:2113/streams/newstream" -H "Content-Type:application/vnd.eventstore.events+json"
```

::::
:::::