\newpage

# Implementing a Common Language and Conditional Access

This section analyzes different approaches to create a common language format between the service of the "Distributed Authentication Mesh". After the analysis, the definition and implementation of the common format enhances the general concept of the Mesh and enables a production-grade software. The PKI and the translators are written in Go, while the Kubernetes Operator is written in C\#. All software is licensed under the **Apache-2.0** license and can be found in the "WirePact" organization: <https://github.com/WirePact>.

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

Certificates have the big advantage: they can be integrity checked via already implemented hashing mechanisms and provide a "trust anchor"^[A trust anchor is a root for all trust in the system.] in the form of a root certificate authority (root CA). Furthermore, if certificates would be used to transmit the users' identity within the authentication mesh, the certificates could also be used to harden the communication between two services. The certificates can enable mutual TLS (mTLS) between communicating services.

But, implementing custom private fields and manipulating that data is cumbersome in various programming languages. In C\# for example, the code to create a simple x509 certificate can span several hundred lines of code. Go^[<https://go.dev/>] on the other hand, has a much better support for manipulating x509 certificates. Since the result of this project should have a good developer experience, using x509 certificates is not be the best solution to solve the communication and integrity issue.

### JSON Web Tokens

A plausible way to encode and protect data is to encode them into a JSON web token (JWT). JWTs are used to encode the user identity in OpenID Connect and OAuth 2.0. A JWT contains three parts: A "header", a "payload" and the "signature" [@RFC7519]. The header identifies which algorithm was used to sign the JWT and can contain other arbitrary information. The payload carries the data that shall be transmitted. The last part of the JWT contains the constructed signature of the header and the payload. This signature is constructed by either a symmetrical or asymmetrical hashing algorithm.

To make use of JWTs in the distributed authentication mesh, another technique for JWTs is used: JSON Web Signatures (JWS). JWS represents data that is secured with a digital signature [@RFC7515]. When considering JWT/JWS for the mesh, a signed token that contains the user ID could be used with key material from the PKI to sign the data and provide a way to securely transmit the data. Since the data is not confidential (typically only a user id), it must be signed only. To help prevent extra round trips, the two extra headers `x5c` and `x5t` can be used to transmit the certificate chain as well as a hash of the certificate to the proxy that is checking the data [RFC7515].

### Using JWT in the Authentication Mesh

After considering the possible transport formats above, we can now analyze the pro and contra arguments. While **structured formats** like YAML and JSON are widely known and easily implemented, they do not offer a built-in mechanism to prevent data manipulation and integrity checking. There are standards that mitigate that matter, but then one can directly use JWT instead of implementing the mechanism by themselves.

**X509** certificates provide an optimal mechanism to transmit extra data with the certificate itself with "private extensions". They could also be used to enable mTLS between services to further harden the communication between participants of the mesh. However, to enable developers to implement custom translators by themselves, x509 certificates are not optimal since the code to manipulate them depends on the used programming language.

**JWT** uses the best of both worlds. They are asynchronously signed with a x509 certificate private key and can transmit the certificate chain as well as a hash of the signing certificate to prevent manipulation. There exist various libraries for many programming languages like Java, C\# or Go. Also, JWTs are already used in similar situations.

## A Public Key Infrastructure as Trust Anchor

The implementation of a PKI is vital to the authentication mesh. The participating translators must be able to fetch valid certificates to sign the JWTs they are transmitting. The PKI can be found at: <https://github.com/WirePact/k8s-pki>. The PKI must fulfill the use cases depicted in {@fig:04_pki_usecases}.

![Use Case Diagram for the PKI](diagrams/04_pki_usecases.puml){#fig:04_pki_usecases}

**Fetch CA Certificate.** Any client must have access to the root CA (certificate authority) to validate the signatures of received JWTs. The signing certificates of the translators are derived by the CA and can therefore be validated if they are authorized to be part of the mesh.

**Sign Certificate Signing Requests.** The participating clients (translators) must be able to create a certificate signing request (CSR) and send them to the PKI. The PKI does create valid certificates that are signed with the root CA and then returns the created certificates to the clients. To validate the certificate chain, an interested party can fetch the public part of the root CA via the other mentioned endpoint and check if the chain is valid.

### "Gin", a Go HTTP Framework

To manage HTTP request to the PKI, the "Gin"^[<https://github.com/gin-gonic/gin>] framework is used. It allows easy management of routing and provides support for middlewares if needed. To set up the CA, the following code enables a web server with the needed routes to `/ca` and `/csr`:

```go
router := gin.Default()

router.GET("ca", api.GetCA)
router.POST("csr", api.HandleCSR)
```

### Prepare the CA

Since the implementation targets a local environment (for development) as well as the Kubernetes environment, the CA and the private key can be stored in multiple ways. For local development, the certificate with the private key is created in the local file system. If the PKI runs within Kubernetes, a `Secret` (encrypted data in Kubernetes) shall be used.

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
	NotBefore: time.Now(),
	NotAfter:  time.Now().AddDate(20, 0, 0),
	IsCA:      true,
	ExtKeyUsage: []x509.ExtKeyUsage{
		x509.ExtKeyUsageClientAuth,
		x509.ExtKeyUsageServerAuth,
	},
	KeyUsage: x509.KeyUsageDigitalSignature |
		x509.KeyUsageCertSign,
	BasicConstraintsValid: true,
}

privateKey, _ := rsa.GenerateKey(rand.Reader, 2048)
publicKey := &privateKey.PublicKey
caCert, err := x509.CreateCertificate(
	rand.Reader,
	ca,
	ca,
	publicKey,
	privateKey)
```

The private key is generated with a cryptographically secure random number generator. After the certificate is generated, it can be encoded and stored in a file or in the Kubernetes secret. The CA certificate is created with a 20-year lifetime. A further improvement to the system could introduce short-lived certificates to mitigate attacks against the CA.

### Deliver the CA

As soon as the preparation process in {@fig:04_pki_prepare_ca} has finished, the CA certificate is ready to be delivered in-memory. This process does not need any special processing power. When a `HTTP GET` request to `/ca` arrives, the PKI will return the public certificate part of the root CA to the caller. This call is used by translators and other participants of the authentication mesh to store the currently valid root CA by themselves and to validate the certificate chain.

```go
context.Header(
	"Content-Disposition",
	`attachment; filename="ca-cert.crt"`)
context.Data(
	http.StatusOK,
	"application/x-x509-ca-cert",
	certificates.GetCA())
```

The only specialty are the data type headers that are set to `application/x-x509-ca-cert`. While they are not necessary, the headers are added for good practice.

The `GetCA()` method itself just returns the public CA certificate:

```go
func GetCA() []byte {
	return pem.EncodeToMemory(
		&pem.Block{Type: "CERTIFICATE", Bytes: ca.Raw})
}
```

The certificates are "PEM" encoded.

### Process Certificate Signing Requests

To be able to sign CSRs, as stated in {@fig:04_pki_usecases}, the PKI must be able to parse and understand CSRs. The PKI supports a `HTTP POST` request to `/csr` that receives a body that contains a CSR.

![Invocation sequence to receive a signed certificate from the PKI.](diagrams/04_pki_handlecsr.puml){#fig:04_pki_handlecsr short-caption="Receive Signed Cert from PKI"}

The sequence in {@fig:04_pki_handlecsr} runs in the PKI. Since the certificate signing request is prepared with the private key of the translator (or the participant of the mesh), no additional keys must be created. The PKI signs the CSR and returns the valid client certificate to the caller. The caller can now sign data with the private key of the certificate and any receiver is able to validate the integrity with the public part of the certificate. Furthermore, the receiver of data can validate the certificate chain with the root CA from the PKI.

If no CSR is attached to the `HTTP POST` call, or if the body contains an invalid CSR, the PKI will return a `HTTP Bad Request` (status code 400) to the sender and abort the call.

### Authentication and Authorization against the PKI

In the current implementation, no authentication and authorization against the PKI are present. Since the current state of the system shall run within the same trust zone, this is not a big threat vector. However, for further implementations and iterations, a mechanism to "authenticate the authenticator" must be implemented.

A security consideration in the distributed authentication mesh is the possibility that _any_ client can fetch a valid certificate from the PKI and then sign _any_ identity within the system. To harden the PKI against unwanted clients, two possible measures can be taken.

**Use a pre-shared key to authorize valid participants.** With a pre-shared key, all valid participants have the proof of knowledge about something that an attacker should have a hard time to come by. In Kubernetes this could be done with an injected secret or a vault software^[Like "HashiVault" <https://www.vaultproject.io/>].

**Use an intermediate certificate for the PKI.** When the PKI itself is not the absolute trust anchor (the root CA), an intermediate certificate could be delivered as a pre-known secret. Participants would then sign their CSRs with that intermediate certificate and therefore proof that they are valid participants of the mesh.

In either way, both measures require the usage of pre-shared or pre-known secrets. Additional options to mitigate this attack vector are not part of this project and shall be investigated in future work.

## Implementing a Translator with a Secure Common Identity

This section describes the definition and the usage of the secure common identity with the example of a translator. The translator uses HTTP Basic Auth (RFC7617 [@RFC7617]) for the username/password combination. The implementation is hosted on <https://github.com/WirePact/k8s-basic-auth-translator>.

### Define the Common Identity

The distributed authentication mesh needs a single source of truth. It is not possible to recover user information out of arbitrary information. As an example, an application that uses multiple services with OIDC and Basic Auth needs a common "base of users". Even if the authentication mesh is not in place, the services need to know which basic authentication credentials they need to use for a specific user.

![Definition of the Common Identity](diagrams/04_translator_identity_definition.puml){#fig:04_translator_identity_definition}

As shown in {@fig:04_translator_identity_definition}, the definition of the common identity is quite simple. The only field that needs to be transmitted is the `subject` of a user. The `subject` (or `sub`) field is defined in the official public claims of a JWT [@RFC7519, sec. 4.1.2].

### Validate and Encode Outgoing Credentials

Any application that shall be part of the authentication mesh must either call the injected forward proxy by itself, or it should respect the `HTTP_PROXY` environment variable, as done by Go and other languages/frameworks. Outgoing communication ("egress") is processed by the envoy proxy with the external authentication mechanism [@buehler:DistAuthMesh]. The following results exist:

- Skip: Do not process any headers or elements in the request
- UserID empty: forbidden request
- Forbidden not empty: for some reason, the request is forbidden
- UserID not empty: the request is allowed

![Skipped/Ignored Egress Request](diagrams/04_translator_egress_skip.puml){#fig:04_translator_egress_skip}

{@fig:04_translator_egress_skip} shows the sequence if the request is "skipped". In this case, skipped means that no headers are consumed nor added. The request is just passed to the destination without any interference. This happens if, in the case of Basic Auth, no `HTTP Authorize` header is added to the request or if another authentication scheme is used (OIDC for example).

![Unauthorized Egress Request](diagrams/04_translator_egress_no_id.puml){#fig:04_translator_egress_no_id}

{@fig:04_translator_egress_no_id} depicts the process when the request contains a correct HTTP header, but the provided username/password combination is not found in the "repository" of the translator. So, no common user ID can be found for the given username and therefore, the provided authentication information is not valid.

![Forbidden Egress Request](diagrams/04_translator_egress_forbidden.puml){#fig:04_translator_egress_forbidden}

In {@fig:04_translator_egress_forbidden}, the HTTP header is present, but corrupted. For example, if the username/password combination was encoded in the wrong format. If this happens, the proxy will reject the request and never bother the destination with incorrect authentication information.

![Processed Egress Request](diagrams/04_translator_egress_ok.puml){#fig:04_translator_egress_ok}

If a request contains the correct HTTP header, the data within is valid and a user can be found with the username/password combination, {@fig:04_translator_egress_ok} shows the process of the request. The translator instructs the forward proxy to consume (i.e. remove) the HTTP authorize header and injects a new custom HTTP header `x-wirepact-identity`. The new header contains a signed JWT that contains the user ID as the subject and the certificate chains as well as a hash of the signing certificate in its headers.

### Validate and Decode an Incoming Identity

## Automate the Authentication Mesh

TODO: describe the operator
