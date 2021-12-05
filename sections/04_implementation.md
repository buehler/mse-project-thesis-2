\newpage

# Implementing a Common Language and Conditional Access

This section analyzes different approaches to create a common language format between the service of the "Distributed Authentication Mesh". After the analysis, the definition and implementation of the common format enhances the general concept of the Mesh and enables a production-grade software. The PKI and the translators are written in Go, while the Kubernetes Operator is written in C\#. All software is licensed under the **Apache-2.0** license and can be found in the "WirePact" organization: https://github.com/WirePact.

## Goals and Non-Goals of the Project

This section provides two tables with functional and non-functional requirements for the project. The mentioned tables extend the requirements of the distributed authentication mesh [@buehler:DistAuthMesh, seq. 4.2]. As mentioned, this project enhances the concept of the distributed authentication mesh by analyzing various ways of transmitting the user identity and defining a meaningful way to transport the identity between participants.

tbl: TODO

- hashing/integrity check
- easy to use

## A Way to Communicate with Integrity

To enable the translators in the distributed authentication mesh to communicate securely, a common format must be used. The format must support a feasible way to prevent modification of the data it transports. The following sections give an overview over the three options that may be used. In the end of the section a comparison shows pro and contra to each option and a decision is made.

### YAML, XML, JSON, and Others

YAML (YAML Ain't Markup Language^[<https://yaml.org/>]), XML (Extensible Markup Language^[<https://www.w3.org/XML/>]), JSON (JavaScript Object Notation^[<https://www.json.org/>]) and other structured data formats such as binary serialized objects in C\# are widely used for data transport. They are typically used to transport structured data in a more or less human-readable form but maintain the possibility to be serialized and deserialized by a machine. Those structures could be used to transport the identity of an authenticated user. However, the formats do not support integrity checks by default.

```yaml
userId: 123456
userName: Test User
```

The example above shows a simple YAML example with an "object" that contains two properties: `userId` and `userName`. These objects can be extended and well typed in programming languages.

There exist approaches like "SecJSON" that enhance JSON with security mechanisms such as data encryption [@santos:SecJSON]. But if the standard of the specifications is used, no integrity check can be performed and the translators of the authentication mesh cannot detect if the data was modified. Thus, using a "simple" structured data format for the transmission of the user identity would not suffice the security requirements of the system.

Similar to "SecJSON", one could add special fields into XML and/or YAML to transmit a hash of the data such that the receiver can validate the data. However, using custom fields do not rely on current standards and are therefore prone to errors implementation wise.

### X509 Certificates

The x509 standard (**RFC5280**) defines how certificates shall be used. Today, the connection over HTTPS is done via TLS and certificate encryption. The fields in a certificate are only partially defined. These "standard extensions" are well-known fields such as the "authority" or alternative names for the subject. In the specification, "private extensions" are another possibility to encode data into certificates [@RFC5280, seq. 4.2]. These extensions could be used to transmit the data needed for the distributed authentication mesh.

Certificates have the big advantage: they can be integrity checked via already implemented hashing mechanisms and provide a "trust anchor"^[A trust anchor is a root for all trust in the system.] in the form of a root certificate authority (root CA). Furthermore, if certificates would be used to transmit the users identity within the authentication mesh, the certificates could also be used to harden the communication between two services. The certificates can enable mutual TLS (mTLS) between communicating services.

But, implementing custom private fields and manipulating that data is cumbersome in various programming languages. In C\# for example, the code to create a simple x509 certificate can span several hundred lines of code. Go^[<https://go.dev/>] on the other hand, has a much better support for manipulating x509 certificates. Since the result of this project should have a good developer experience, using x509 certificates is not be the best solution to solve the communication and integrity issue.

### JSON Web Tokens

A plausible way to encode and protect data is to encode them into a JSON web token (JWT). JWTs are used to encode the user identity in OpenID Connect and OAuth 2.0. A JWT contains three parts: A "header", a "payload" and the "signature" [@RFC7519]. The header identifies which algorithm was used to sign the JWT and can contain other arbitrary information. The payload carries the data that shall be transmitted. The last part of the JWT contains the constructed signature of the header and the payload. This signature is constructed by either a symmetrical or asymmetrical hashing algorithm.

To make use of JWTs in the distributed authentication mesh, another technique for JWTs is used: JSON Web Signatures (JWS). JWS represents data that is secured with a digital signature [@RFC7515]. When considering JWT/JWS for the mesh, a signed token that contains the user ID could be used with key material from the PKI to sign the data and provide a way to securely transmit the data. Since the data is not confidential (typically only a user id), it must be signed only. To help prevent extra round trips, the two extra headers `x5c` and `x5t` can be used to transmit the certificate chain as well as a hash of the certificate to the proxy that is checking the data [RFC7515].

### Using JWT in the Authentication Mesh

After considering the possible transport formats above, we can now analyze the pro and contra arguments. While **structured formats** like YAML and JSON are widely known and easily implemented, they do not offer a builtit mechanism to prevent data manipulation and integrity checking. There are standards that mitigate that matter, but then one can directly use JWT instead of implementing the mechanism by themselves.

**X509** certificates provide an optimal mechanism to transmit extra data with the certificate itself with "private extensions". They could also be used to enable mTLS between services to further harden the communication between participants of the mesh. However, to enable developers to implement custom translators by themselves, x509 certificates are not optimal since the code to manipulate them depends on the used programming language.

**JWT** uses the best of both worlds. They are asynchronously signed with a x509 certificate private key and can transmit the certificate chain as well as a hash of the signing certificate to prevent manipulation. There exist various libraries for many programming languages like Java, C\# or Go. Also, JWTs are already used in similar situations.

## A Public Key Infrastructure as Trust Anchor

The implementation of a PKI is vital to the authentication mesh. The participating translators must be able to fetch valid certificates to sign the JWTs they are transmitting. The PKI can be found at: https://github.com/WirePact/k8s-pki. The PKI must fulfill the following use cases:

**Provide the root certificate authority (CA).** Any client must have access to the root CA to validate the signatures of received JWTs. The signing certificates of the translators are derived by the CA and can therefore be validated if they are authorized to be part of the mesh.

**Provide certificates to participants.** The participating clients (translators) must be able to create a certificate signing request (CSR) and send them to the PKI. The PKI does create valid certificates that are signed with the root CA and then returns the created certificates to the clients. To validate the certificate chain, an interested party can fetch the public part of the root CA via the other mentioned endpoint and check if the chain is valid.

### "Gin", a Go HTTP Framework

To manage HTTP request to the PKI, the "Gin"^[<https://github.com/gin-gonic/gin>] framework is used. It allows easy management of routing and provides support for middlewares if needed. To set up the CA, the following code enables a web server with the needed routes to `/ca` and `/csr`:

```go
router := gin.Default()

router.GET("ca", api.GetCA)
router.POST("csr", api.HandleCSR)
```

### Prepare the CA

Since the implementation targets a local environment (for development) as well as the Kuberentes environment, the CA and the private key can be stored in multiple ways. For local development, the certificate with the private key is created in the local file system. If the PKI runs within Kubernetes, a `Secret` (encrypted data in Kubernetes) shall be used.

![Prepare CA method in the PKI on Startup](diagrams/04_pki_prepare_ca.puml){#fig:04_pki_prepare_ca}

During the startup of the PKI, a "prepare CA" method runs and checks if all needed objects are in place. {@fig:04_pki_prepare_ca} shows the invocation sequence of the method. If the PKI runs in "local mode" (meaning it is used for local development and has no access to Kubernetes), the CA certificate and private key are stored in the local file system. Otherwise, the Kubernetes secret is prepared and the certificate loaded/created from the secret.

To create a new CA certificate, the following code can be used:

```go
ca := &x509.Certificate{
	SerialNumber: big.NewInt(getNextSerialnumber()),
	Subject: pkix.Name{
		Organization: []string{"WirePact PKI CA"},
		Country:      []string{"Kubernetes"},
		CommonName:   "PKI",
	},
	NotBefore:             time.Now(),
	NotAfter:              time.Now().AddDate(20, 0, 0),
	IsCA:                  true,
	ExtKeyUsage:           []x509.ExtKeyUsage{x509.ExtKeyUsageClientAuth, x509.ExtKeyUsageServerAuth},
	KeyUsage:              x509.KeyUsageDigitalSignature | x509.KeyUsageCertSign,
	BasicConstraintsValid: true,
}

privateKey, _ := rsa.GenerateKey(rand.Reader, 2048)
publicKey := &privateKey.PublicKey
caCert, err := x509.CreateCertificate(rand.Reader, ca, ca, publicKey, privateKey)
```

The private key is generated with a cryptographically secure random number generator. After the certificate is generated, it can be encoded and stored in a file or in the Kubernetes secret.

### Deliver the CA

### Process Certificate Signing Requests

## Implementing a Secure Common Identity

This section describes the process of implementing the common language format and the needed systems for Kubernetes. To use the distributed authentication mesh, a public key infrastructure as well as translators that can validate and interpret JWTs are needed. The implementation is assumed to run on Kubernetes, but the concepts are adaptable to other cloud environments or platforms as well.

TODO: this describes the translators that use JWT, use the example of the basic auth translator

## Automate the Authentication Mesh

TODO: describe the operator
