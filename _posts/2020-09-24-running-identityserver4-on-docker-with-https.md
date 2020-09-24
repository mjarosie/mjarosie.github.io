---
categories: ["Dev"]
tags: ["Docker", "HTTPS", "certificates", "IdentityServer4", "oAuth", ".NET"]
date:  2020-09-24
layout: post
title:  "Securing an API while running IdentityServer4 on Docker with HTTPS enabled locally"
---

Recently I've been trying to spin up an instance of [IdentityServer4](https://identityserver4.readthedocs.io/en/latest/) which would protect an example API with [Client Credentials Flow](https://auth0.com/docs/flows/client-credentials-flow) - just to get my head around it.

What I wanted to achieve:

- communication between services should work the same way locally as in production (hence, it should be secure - going through HTTPS)
- services should be Dockerised so that the developer setup is as easy as possible (I also wanted to try out the new Docker Desktop with WSL 2 integration [finally available for Windows Home users](https://www.docker.com/blog/docker-desktop-for-windows-home-is-here/))

The easiest way to start with IdentityServer4 is to go through the [official quickstart tutorial](https://identityserver4.readthedocs.io/en/latest/quickstarts/1_client_credentials.html), but even though the examples leverage HTTPS, putting it into Docker turned out to be way much trickier than I initially expected. I'm not the first one who has run into this issue (for instance see [here](https://stackoverflow.com/questions/43911536), [here](https://stackoverflow.com/questions/44481538) or [here](https://stackoverflow.com/questions/53468074). Or [here](https://stackoverflow.com/questions/61997345). Or [here](https://stackoverflow.com/questions/62065739)...), but I couldn't find any fully working reproducible solution/explanation on the Internet, so here's the post.

## The localhost's gone!

We've got 3 services involved:

- IdentityServer4 (aka [Identity Provider](https://en.wikipedia.org/wiki/Identity_provider))
- API
- Client (user of the API)

You've probably seen the diagram describing communication between these services (or some version of it) if you've already worked with Client Credentials Flow:

![Client Credentials Flow sequence diagram](/assets/2020-09-24-running-identityserver4-on-docker-with-https/client-credentials-flow-sequence-diagram.png)

In order for the Client to be able to retrieve a resource from the API, first it needs to obtain a valid access token (shown as `(*)` in the above diagram) from the Identity Provider that's trusted by the API.

When you're just running these services on your host machine, they all refer to each other by `localhost` addresses:

![localhost network](/assets/2020-09-24-running-identityserver4-on-docker-with-https/localhost.png)

The moment you put IdentityServer and API in the Docker container and spin up both of these using docker-compose, [by default a new network is created](https://docs.docker.com/compose/networking/). The new network is separate from your `localhost`. From your host machine you can still refer to these services as `localhost`, but within the Docker network services need to use container names (`identity-server` and `web-api` in this specific scenario) to speak to each other:

![Docker network](/assets/2020-09-24-running-identityserver4-on-docker-with-https/docker.png)

## First attempt - using dotnet dev-certs tool

It's all hunky-dory when you're using HTTP for cross-service communication (we don't want that), but not when certificates come into play.

Microsoft docs describe [how to use HTTPS in Docker Compose](https://docs.microsoft.com/en-us/aspnet/core/security/docker-compose-https?view=aspnetcore-3.1) but that doesn't work when services need to refer to each other by anything else than `localhost`, since dev certificates that are accessed with `dotnet dev-certs` tool [are issued for localhost only](https://docs.microsoft.com/en-us/aspnet/core/getting-started/?view=aspnetcore-3.1&tabs=windows#trust-the-development-certificate):

![Docker network with default localhost certificates](/assets/2020-09-24-running-identityserver4-on-docker-with-https/docker-localhost-certs.png)

Running a variation of this command: `dotnet dev-certs https -ep %USERPROFILE%\.aspnet\https\aspnetapp.pfx -p password` (see Microsoft docs link above for details) will export a `.pfx` certificate. You then need to put it in your Docker container and redirect Kestrel to use it. After setting up environment variables your `docker-compose` could look something along these lines:

```yaml
version: '3'

services:
  identity-server:
    build: ./src/IdentityServer
    ports:
      - '5001:5001'
    volumes:
      - ./src/IdentityServer:/root/IdentityServer:cached
      - ${USERPROFILE}/.aspnet/https:/https/
    environment:
      - ASPNETCORE_URLS="https://+;"
      - ASPNETCORE_HTTPS_PORT=5001
      - ASPNETCORE_Kestrel__Certificates__Default__Password=password
      - ASPNETCORE_Kestrel__Certificates__Default__Path=/https/aspnetapp.pfx
  web-api:
    build: ./src/Api
    ports:
        - '6001:6001'
    volumes:
      - ./src/Api:/root/Api:cached
      - ${USERPROFILE}/.aspnet/https:/https/
    environment:
      - ASPNETCORE_URLS="https://+;"
      - ASPNETCORE_HTTPS_PORT=6001
      - ASPNETCORE_Kestrel__Certificates__Default__Password=password
      - ASPNETCORE_Kestrel__Certificates__Default__Path=/https/aspnetapp.pfx
```

Unfortunately, it's going to work only as long as these Docker services won't start calling each other. You will run into a certificate validation issues when the API tries to securely connect to the IdentityServer to validate the token (if you're lost - refer back to the Client Credentials Flow diagram at the top of this post). Even though the client connects to `localhost` (see the network diagram above) - API and IdentityServer need to refer to each other as `identity-service` and `web-api` respectively (those are the container names defined in docker-compose). The API won't be able to reach IdentityServer under `localhost:5001` anymore, and when it tries to connect to `identity-service:5001` the certificate validation fails, since the certificate wasn't issued for `identity-service`, but for `localhost`!

Another problem is the name of the issuer (the Authority setting in [the API configuration](https://identityserver4.readthedocs.io/en/latest/quickstarts/1_client_credentials.html#configuration)). By default IdentityServer sees itself as `localhost:5001` and when you run it and retrieve the [discovery endpoint](https://identityserver4.readthedocs.io/en/latest/endpoints/discovery.html) ([https://localhost:5001/.well-known/openid-configuration]) it will respond with a JSON with `issuer` value being equal to `https://localhost:5001`. But when the API calls the IdentityServer to check if the token is valid, by default it checks if the URL of the identity provider (`https://identity-service:5001`) is the same as the name of the token issuer that's extracted from the token (`https://localhost:5001`) - which is not the case!

## It is cert to work now!

To make it work nicely, we have to manually issue certificates, so that `API` service doesn't run into certificate validation issues when calling `IdentityServer`:

![Docker network with correct certificates](/assets/2020-09-24-running-identityserver4-on-docker-with-https/docker-correct-certs.png)

First we need to issue the certificate with CN ([Common Name, also known as Fully Qualified Domain Name (FQDN)](https://knowledge.digicert.com/solution/SO7239.html)) having entries for both `identity-server` and `localhost` for IdentityServer, and `web-api` and `localhost` for the API. I've done it [this way](https://github.com/mjarosie/IdentityServerDockerHttpsDemo/blob/master/generate_self_signed_cert.ps1) by issuing a self-signed root CA certificate which is then used to sign certificates for both IdentityServer and API using [New-SelfSignedCertificate](https://docs.microsoft.com/en-us/powershell/module/pkiclient/new-selfsignedcertificate) and then exported with [Export-PfxCertificate](https://docs.microsoft.com/en-us/powershell/module/pkiclient/export-pfxcertificate) and [Export-Certificate](https://docs.microsoft.com/en-us/powershell/module/pkiclient/export-certificate):

```powershell
$testRootCA = New-SelfSignedCertificate -Subject $rootCN -KeyUsageProperty Sign -KeyUsage CertSign -CertStoreLocation Cert:\LocalMachine\My
$identityServerCert = New-SelfSignedCertificate -DnsName $identityServerCNs -Signer $testRootCA -CertStoreLocation Cert:\LocalMachine\My
$webApiCert = New-SelfSignedCertificate -DnsName $webApiCNs -Signer $testRootCA -CertStoreLocation Cert:\LocalMachine\My

$password = ConvertTo-SecureString -String "password" -Force -AsPlainText

$rootCertPathPfx = "certs"
$identityServerCertPath = "src/IdentityServer/certs"
$webApiCertPath = "src/Api/certs"

Export-PfxCertificate -Cert $testRootCA -FilePath "$rootCertPathPfx/aspnetapp-root-cert.pfx" -Password $password | Out-Null
Export-PfxCertificate -Cert $identityServerCert -FilePath "$identityServerCertPath/aspnetapp-identity-server.pfx" -Password $password | Out-Null
Export-PfxCertificate -Cert $webApiCert -FilePath "$webApiCertPath/aspnetapp-web-api.pfx" -Password $password | Out-Null

# Export .cer to be converted to .crt to be trusted within the Docker container.
$rootCertPathCer = "certs/aspnetapp-root-cert.cer"
Export-Certificate -Cert $testRootCA -FilePath $rootCertPathCer -Type CERT | Out-Null

# Trust it on your host machine.
$store = New-Object System.Security.Cryptography.X509Certificates.X509Store "Root","LocalMachine"
$store.Open("ReadWrite")
$store.Add($testRootCA)
$store.Close()
```

We need to orchestrate `docker-compose` to put these certificates in containers and set up environment variables correctly:

```yaml
version: '3'

services:
  identity-server:
    build: ./src/IdentityServer
    ports:
      - '5001:5001'
    volumes:
      - ./src/IdentityServer:/root/IdentityServer:cached
      - ./src/IdentityServer/certs:/https/
      - type: bind # Using a bind volume as only this single file from `certs` directory should end up in the container.
        source: ./certs/aspnetapp-root-cert.cer
        target: /https-root/aspnetapp-root-cert.cer
    environment:
      - ASPNETCORE_URLS="https://+;"
      - ASPNETCORE_HTTPS_PORT=5001
      - ASPNETCORE_Kestrel__Certificates__Default__Password=password
      - ASPNETCORE_Kestrel__Certificates__Default__Path=/https/aspnetapp-identity-server.pfx

  web-api:
    build: ./src/Api
    ports:
        - '6001:6001'
    volumes:
      - ./src/Api:/root/Api:cached
      - ./src/Api/certs:/https/
      - type: bind # Using a bind volume as only this single file from `certs` directory should end up in the container.
        source: ./certs/aspnetapp-root-cert.cer
        target: /https-root/aspnetapp-root-cert.cer
    environment:
      - ASPNETCORE_URLS="https://+;"
      - ASPNETCORE_HTTPS_PORT=6001
      - ASPNETCORE_Kestrel__Certificates__Default__Password=password
      - ASPNETCORE_Kestrel__Certificates__Default__Path=/https/aspnetapp-web-api.pfx
```

Then we need to trust IdentityServer's certificate from the API service by trusting the root CA certificate within the Docker container (as described [here](https://askubuntu.com/questions/73287/how-do-i-install-a-root-certificate/94861#94861) - I had to take an extra step of converting between `.cer` and `.crt` formats as `Export-Certificate` supports only the former and Ubuntu - only the latter, see [docker-entrypoint.sh](https://github.com/mjarosie/IdentityServerDockerHttpsDemo/blob/master/src/Api/docker-entrypoint.sh):

```bash
openssl x509 -inform DER -in /https-root/aspnetapp-root-cert.cer -out /https-root/aspnetapp-root-cert.crt
cp /https-root/aspnetapp-root-cert.crt /usr/local/share/ca-certificates/
update-ca-certificates
```

IdentityServer needs to refer to itself (IssuerUri value) as `https://identity-server:5001`, [see the configuration](https://github.com/mjarosie/IdentityServerDockerHttpsDemo/blob/master/src/IdentityServer/Startup.cs#L29):

```c#
public void ConfigureServices(IServiceCollection services)
{
    var builder = Environment.IsDevelopment() ?
        services.AddIdentityServer(x =>
        {
            x.IssuerUri = "https://identity-server:5001";
        }) :
        services.AddIdentityServer();
    builder
        .AddInMemoryApiScopes(Config.ApiScopes)
        .AddInMemoryClients(Config.Clients);
}
```
API needs to refer to the Authority (IdentityServer) as `https://identity-server:5001`, [see the configuration](https://github.com/mjarosie/IdentityServerDockerHttpsDemo/blob/master/src/Api/Startup.cs#L28):

```c#
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers();

    services.AddAuthentication("Bearer")
        .AddJwtBearer("Bearer", options =>
        {
            options.Authority = "https://identity-server:5001";

            options.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateAudience = false
            };

        });
}
```

Client [must not be validating the issuer name](https://github.com/mjarosie/IdentityServerDockerHttpsDemo/blob/master/src/Client/Program.cs#L20) when retrieving discovery document as names won't match (`identity-server` vs. `localhost`):

```c#
var disco = await client.GetDiscoveryDocumentAsync(
    new DiscoveryDocumentRequest
    {
        Address = "https://localhost:5001",
        Policy =
        {
            ValidateIssuerName = false
        },
    });
```

Fully working example can be found [in this repository](https://github.com/mjarosie/IdentityServerDockerHttpsDemo).