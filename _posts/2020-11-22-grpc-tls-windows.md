---
layout: post
title: Setting up SSL/TLS channels in Grpc/C# using Certificates from the Windows Certificate Store
author: ema
date: 2020-11-22
---
Knowing how SSL/TLS  really works is something developers often treat like knowing how dishwashers work. You try to avoid it until some reality forces you to deal with it. Once figured out it appears rather quite forward (apart from some details) but you still gain a lot of respect for the people who invented it.

The example code is located [here](https://github.com/emahler/grpc-secure-greeter/).

The use of [X509Certificate2](https://docs.microsoft.com/en-us/dotnet/api/system.security.cryptography.x509certificates.x509certificate2?view=netframework-4.8) objects from the certificate store in Windows to set up a SSL/TLS channel in a grpc microservice is remarkably little documented on the interwebs. This post is about Grpc.Core (the original grpc for C#) which, as opposed to grpc-dotnet (the .NET core implementation). does not provide any infrastructure to use .NET certificates directly as SSLCredentials for TLS. Unfortunately, there are no plans to fix this shortcoming (according to grpc issue [8978](https://github.com/grpc/grpc/issues/8978)). Obviously pull requests are welcome, but as a complete SSL/TLS noob (until at least until a few days ago) I might not be the ideal grpc contributer when it comes to copy-pasting security-related code from stack overflow. But the actual problem is that grpc supports the .NET Framwork 4.5 when the certificate infrastructure was vastly different from now.

## Authentication Basics

In the typical use cases only the **client authenticates the server**. In order to determine that the server really represents the domain address the client wants to connect to, the server must proof it has a valid certificate during the SSL/TLS handshake. In short, this is a [negotiation ceremony](http://www.moserware.com/2009/06/first-few-milliseconds-of-https.html) in which the server can show it has the matching private key for its public key without actually revealing the private key using the mathemagic of one-way functions. The most fascinating part is the Diffie-Hellman key exchange that allows exchanging a secret key used for the subsequent symmetrical encryption over an unsecure network instead of using a secure phone line or mail coach. However, to be sure of the server's identity, its certificate must have been verified (digitally signed) by a root certification authority (CA) that is trusted by the client, or at least it must have been verified by an intermediate authority trusted by a root certification authority ("chain of trust").

In more advanced use cases the server can authenticate the client as well in which case the same is done the other way round using a signed certificate from the client. This is referred to as **mutual authentication**. However, in most cases this is done using domain-specific user-id/password mechanisms or Kerberos tokens for Windows authentication mechanisms.

## grpc Authentication using TLS

In order to create the grpc channel, the server needs to have the public key and the private key of its certificate whereas the client connection must provide its trusted root certificates (or at least the one on top of the server's certificate chain). In very basic setups inside a single organization the public key of the server's certificate can be used directly, if copying files around is acceptable.

On Windows, the certificates are managed in the Certificate Store and can be examined using the Microsoft Management Console (certlm, certmgr). Typically the distribution of certificates, including self-signed Enterprise root CA is done behind the scenes by the organization.

For our microservices we want to use the certificates from this Certificate Store to set up the SSL/TLS credentials. On Windows, the trusted root certificates can be found in the Certificate Store ("Manage user certificates" or certmgr in the Microsoft Management Console) under "Third-Party Root Certification Authorities".

### On the Server

In order to create the grpc Server, the [SslServerCredentials](https://grpc.github.io/grpc/csharp/api/Grpc.Core.SslServerCredentials.html) object must be provided which requires

- the certificate (containing just the public key) including all certificates in the trust chain 
- and, separately, the private key 

Both these strings must be in the PEM format. So the main task here is to [convert](https://github.com/emahler/grpc-secure-greeter/blob/ce4bca7f61b5e50c5c26f623a7d9badbcbf90e50/Certificates/CertificateUtils.cs#L146-L160) the relevant certificates from the Certificate Store ("Manage computer certificates" or certlm in the Microsoft Management Console) which contain both private key and public keys to the PEM format.

### On the Client

In the typical use-case where the client only authenticates the server the client only has to provide the root certificates They are used to make sure the server's certificate chain contains one of the certificates trusted by the client. Trusted root certificates are found in the certificate store under "Third-Party Root Certification Authority". In order to create the client's [SslCredentials](https://grpc.github.io/grpc/csharp/api/Grpc.Core.SslCredentials.html) object these root certificates must also be converted to the PEM format.

If the server also has to authenticate the client (mutual TLS), the SslCredentials require the [KeyCertificatePair](https://grpc.github.io/grpc/csharp/api/Grpc.Core.KeyCertificatePair.html) of the client's signed certificate. In this case the private key must be extracted in the same way as on the server.

## Using Certificates from the Certificate Store

Getting the certificates is the easy part. For example, the root certificates are in the [Certificate Store](https://docs.microsoft.com/en-us/dotnet/api/system.security.cryptography.x509certificates.x509store?view=netcore-3.1) created like this:

```c#
X509Store store = new X509Store (StoreName.Root, StoreLocation.CurrentUser);
```

The server side certificates used to authenticate the server to clients are usually in the Local Machine Store. With the appropriate privileges (just run it as administrator;-) they can be extracted using 

```c#
X509Store store = new X509Store (StoreName.My, StoreLocation.LocalMachine);
```

The client side certificates (the ones that include the private key used to authenticate the client) are taken from the current user store:

```c#
X509Store store = new X509Store (StoreName.My, StoreLocation.CurrentUser);
```

Things normally start to go wrong when the private key is extracted. For example,

The user that runs the process does **not have the appropriate privileges to access the private key** in which case `X509Certificate2.GetRSAPrivateKey()` throws a CryptographicException: "Keyset does not exist". 

The **private key is marked as non-exportable** in which case our the `RSA.ExportParameters(true)` method throws a CryptographicException: "The requested operation is not supported" or, probably if the certificate is associated with a smart card (PKI) "Invalid type specified".

When opening the client certificate an **authentication dialog can pop up** asking for credentials, such as the PKI smart card associated with the user. This often spoils more than just your unit tests.

Various other things could potentially go wrong, such as a [different private key format](https://www.pkisolutions.com/accessing-and-using-certificate-private-keys-in-net-framework-net-core/). So make sure you really want to go down that rabbit hole before you consider mutual TLS.

## Some Caveats
### Host Address in the Certificate
The symptom for this issue is the **Failed to pick subchannel** exception:

`Grpc.Core.RpcException: Status(StatusCode="Unavailable", Detail="failed to connect to all addresses", DebugException="Grpc.Core.Internal.CoreErrorDetailException: {"created":"@1604912567.530000000","description":"Failed to pick subchannel","file":"T:\src\github\grpc\workspace_csharp_ext_windows_x64\src\core\ext\filters\client_channel\client_channel.cc","file_line":4166,"referenced_errors":[{"created":"@1604912567.530000000","description":"failed to connect to all addresses","file":"T:\src\github\grpc\workspace_csharp_ext_windows_x64\src\core\ext\filters\client_channel\lb_policy\pick_first\pick_first.cc","file_line":398,"grpc_status":14}]}")`

This is probably a real beginner's mistake, or at least the first of mine! The server address must appear in the Common Name or, more commonly in the Server Alternative Name field of the certificate. This must correspond with the host name used for the connection. So "localhost" and a proper certificate found on the machine containing the string DNS Name=machinename.domain-name.com in the subject alternative name won't work! So both the client and the server must specify the proper host name which must be the same as in the certificate.

### VPN

DNS-Related errors sometimes come down to the VPN not being started *before* the server is started. One such error message is

`Grpc.Core.RpcException: Status(StatusCode="Unavailable", Detail="DNS resolution failed for service: scatola.my-domain.com:5151", DebugException="Grpc.Core.Internal.CoreErrorDetailException: {"created":"@1605826338.522000000","description":"Resolver transient failure","file":"T:\src\github\grpc\workspace_csharp_ext_windows_x86\src\core\ext\filters\client_channel\resolving_lb_policy.cc","file_line":214,"referenced_errors":[{"created":"@1605826338.522000000","description":"DNS resolution failed for service: scatola.my-domain.com:5151","file":"T:\src\github\grpc\workspace_csharp_ext_windows_x86\src\core\ext\filters\client_channel\resolver\dns\c_ares\dns_resolver_ares.cc","file_line":378,"grpc_status":14,"referenced_errors":[{"created":"@1605826338.522000000","description":"C-ares status is not ARES_SUCCESS qtype=A name=scatola.my-domain.com is_balancer=0: DNS server returned answer with no data","file":"T:\src\github\grpc\workspace_csharp_ext_windows_x86\src\core\ext\filters\client_channel\resolver\dns\c_ares\grpc_ares_wrapper.cc","file_line":287,"referenced_errors":[{"created":"@1605826338.522000000","description":"C-ares status is not ARES_SUCCESS qtype=AAAA name=scatola.my-domain.com is_balancer=0: DNS server returned answer with no data","file":"T:\src\github\grpc\workspace_csharp_ext_windows_x86\src\core\ext\filters\client_channel\resolver\dns\c_ares\grpc_ares_wrapper.cc","file_line":287}]}]}]}")`

### Very long wait until the call is received by the server
"Very long" in this case is roughly a couple of minutes. In some cases there can also be a "DNS resolution failed for service" exception. This is probably related to a VPN or a disconnected VPN that was previously connected. Setting the following environment variable usually resolves the issue:

set GRPC_DNS_RESOLVER=native

If this is an issue on a production server rather than on a development notebook, something needs to be done!

## Summary

Use a certificate on the server to enable transport-layer security (SSL/TLS) and avoid man-in-the-middle attacks.

Mutual TLS authentication on the other hand also requires a certificate on each client that is accessible to the running process and that has a private key marked as exportable. The extra administrative and development effort (due to the number of things that can go wrong), is probably not justified by the additional benefit of mutual TLS. For those who can use it, the grpc-dotnet implementation has a simple way to add client certificates to the HttpClientHandler, which can probably even use certificates with non-exportable keys. At some point we will learn about grpc-dotnet authentication as well...