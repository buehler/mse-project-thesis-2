\newpage

# Implementing a Common Language and Conditional Access {#sec:implementation}

This section analyzes different approaches to create a common language format between the service of the "Distributed Authentication Mesh". After the analysis, the definition and implementation of the common format enhances the general concept of the Mesh and enables a production-grade software. The PKI and the translators are written in Go, while the Kubernetes Operator is written in C\#. All software is licensed under the **Apache-2.0** license and can be found in the "WirePact" organization: <https://github.com/WirePact>.

## Goals and Non-Goals of the Project

As mentioned, this project enhances the concept of the distributed authentication mesh by analyzing various ways of transmitting the user identity and defining a meaningful way to transport the identity between participants. As such, the goals and non-goals of this project remain the same as in the past work of the distributed authentication mesh [@buehler:DistAuthMesh, ch. 4].

```{.include}
tables/04_functional_requirements.md
```

{@tbl:functional-requirements} shows the functional requirements for this project.

```{.include}
tables/04_non_functional_requirements.md
```

{@tbl:non-functional-requirements} shows the non-functional requirements for this project.

{@tbl:functional-requirements} and {@tbl:non-functional-requirements} extend the existing requirements from the past work in [@buehler:DistAuthMesh]. In general, the system must not be less secure than the current existing security standards. The definition of the common language format must contain a way to check the integrity of the transmitted data and the translators must not interfere with the data stream and only modify HTTP headers.

## A Way to Communicate with Integrity

To enable the translators in the distributed authentication mesh to communicate securely, a common format must be used [@buehler:DistAuthMesh]. The format must support a feasible way to prevent modification of the data it transports. The following sections give an overview over the three options that may be used. In the end of the section a comparison shows pro and contra to each option and a decision is made.

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

The x509 standard (**RFC5280**) defines how certificates shall be used. Today, the connection over HTTPS is done via TLS and certificate encryption. The fields in a certificate are only partially defined. These "standard extensions" are well-known fields such as the "authority" or alternative names for the subject. In the specification, "private extensions" are another possibility to encode data into certificates [@RFC5280, ch. 4]. These extensions could be used to transmit the data needed for the distributed authentication mesh.

Certificates have the big advantage: they can be integrity checked via already implemented hashing mechanisms and provide a "trust anchor"^[A trust anchor is a root for all trust in the system.] in the form of a root certificate authority (root CA). Furthermore, if certificates would be used to transmit the users' identity within the authentication mesh, the certificates could also be used to harden the communication between two services. The certificates can enable mutual TLS (mTLS) between communicating services.

But, implementing custom private fields and manipulating that data is cumbersome in various programming languages. In C\# for example, the code to create a simple x509 certificate can span several hundred lines of code. Go^[<https://go.dev/>] on the other hand, has a much better support for manipulating x509 certificates. Since the result of this project should provide a good developer experience, using x509 certificates is not be the best solution to solve the communication and integrity issue. If future work implements mTLS to harden the communication between services, it may be feasible to transmit the users' identity within the used certificates.

### JSON Web Tokens

A plausible way to encode and protect data is to encode them into a JSON web token (JWT). JWTs are used to encode the user identity in OpenID Connect and OAuth 2.0. A JWT contains three parts: A "header", a "payload" and the "signature" [@RFC7519]. The header identifies which algorithm was used to sign the JWT and can contain other arbitrary information. The payload carries the data that shall be transmitted. The last part of the JWT contains the constructed signature of the header and the payload. This signature is constructed by either a symmetrical or asymmetrical hashing algorithm.

To make use of JWTs in the distributed authentication mesh, another technique for JWTs is used: JSON Web Signatures (JWS). JWS represents data that is secured with a digital signature [@RFC7515]. When considering JWT/JWS for the mesh, a signed token that contains the user ID could be used with key material from the PKI to sign the data and provide a way to securely transmit the data. Since the data is not confidential (typically only a user ID), it must be signed only. To help prevent extra round trips, the two extra headers `x5c` and `x5t` can be used to transmit the certificate chain as well as a hash of the certificate to the proxy that is checking the data [@RFC7515].

In contrast to the above-mentioned "SecJSON", a JWT is well-defined by an RFC. SecJSON enables encrypted data within JSON but does lack the means of integrity checking. A JWT does not encrypt the data but uses JWS for hashing and signing of the data to prevent modification of the data.

### Using JWT in the Authentication Mesh

After considering the possible transport formats above, we can now analyze the pro and contra arguments. While **structured formats** like YAML and JSON are widely known and easily implemented, they do not offer a built-in mechanism to prevent data manipulation and integrity checking. There are standards that mitigate that matter, but then one can directly use JWT instead of implementing the mechanism by themselves.

**X509** certificates provide an optimal mechanism to transmit extra data with the certificate itself with "private extensions". They could also be used to enable mTLS between services to further harden the communication between participants of the mesh. However, to enable developers to implement custom translators by themselves, x509 certificates are not optimal since the code to manipulate them heavily depends on the used programming language.

**JWT** uses the best of both worlds. They are asynchronously signed with a x509 certificate private key and can transmit the certificate chain as well as a hash of the signing certificate to prevent manipulation. There exist various libraries for many programming languages like Java, C\# or Go. Also, JWTs are already used in similar situations like the ID tokens for OpenID Connect.

## A Public Key Infrastructure as Trust Anchor

The implementation of a PKI is vital to the authentication mesh. The participating translators must be able to fetch valid certificates to sign the JWTs they are transmitting. The PKI can be found at: <https://github.com/WirePact/k8s-pki>. The PKI must fulfill the use cases depicted in {@fig:04_pki_usecases}.

![Use Case Diagram for the PKI](diagrams/04_pki_usecases.puml){#fig:04_pki_usecases}

**Fetch CA Certificate.** Any translator must have access to the root CA (certificate authority) to validate the signatures of received JWTs. The signing certificates of the translators are derived by the CA and can therefore be validated if they are authorized to be part of the mesh.

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

The certificates are "PEM"^[PEM Encoding: <https://de.wikipedia.org/wiki/Privacy_Enhanced_Mail>] encoded.

![Deliver public certificate invocation](diagrams/04_pki_deliver_ca.puml){#fig:04_pki_deliver_ca}

The process to deliver the CA is not very complex, as is shown in {@fig:04_pki_deliver_ca}. As soon as the startup process in {@fig:04_pki_prepare_ca} is finished, the PKI can return the public part of the certificate to any client that sends a `HTTP GET` to `/ca` of the PKI.

### Process Certificate Signing Requests (CSR)

To be able to sign CSRs, as stated in {@fig:04_pki_usecases}, the PKI must be able to parse and understand CSRs. The PKI supports a `HTTP POST` request to `/csr` that receives a body that contains a CSR.

![Invocation sequence to receive a signed certificate from the PKI.](diagrams/04_pki_handlecsr.puml){#fig:04_pki_handlecsr short-caption="Receive Signed Cert from PKI"}

The sequence in {@fig:04_pki_handlecsr} runs in the PKI. Since the certificate signing request is prepared with the private key of the translator (or the participant of the mesh), no additional keys must be created. The PKI signs the CSR and returns the valid client certificate to the caller. The caller can now sign data with the private key of the certificate and any receiver is able to validate the integrity with the public part of the certificate. Furthermore, the receiver of data can validate the certificate chain with the root CA from the PKI.

If no CSR is attached to the `HTTP POST` call, or if the body contains an invalid CSR, the PKI will return a `HTTP Bad Request` (status code 400) to the sender and abort the call.

### Authentication and Authorization against the PKI

In the current implementation, no authentication and authorization against the PKI exist. Since the current state of the system shall run within the same trust zone, this is not a big threat vector. However, for further implementations and iterations, a mechanism to "authenticate the authenticator" must be implemented.

A security consideration in the distributed authentication mesh is the possibility that _any_ client can fetch a valid certificate from the PKI and then sign _any_ identity within the system. To harden the PKI against unwanted clients, two possible measures can be taken.

**Use a pre-shared key to authorize valid participants.** With a pre-shared key, all valid participants have the proof of knowledge about something that an attacker should have a hard time to come by. In Kubernetes this could be done with an injected secret or a vault software^[Like "HashiVault" <https://www.vaultproject.io/>].

**Use an intermediate certificate for the PKI.** When the PKI itself is not the absolute trust anchor (the root CA), an intermediate certificate could be delivered as a pre-known secret. Participants would then sign their CSRs with that intermediate certificate and therefore proof that they are valid participants of the mesh.

In either way, both measures require the usage of pre-shared or pre-known secrets. Additional options to mitigate this attack vector are not part of this project and shall be investigated in future work.

## Provide a Translator Base

To enable developers to create translators easily, the GitHub organization "WirePact"^[WirePact: The development name for the distributed authentication mesh.] provides a translator base package written in Go. This package contains helpers and utilities that are needed in a translator and further provide a developer friendly way to implement a translator. The package is available on the GitHub repository <https://github.com/WirePact/go-translator>.

### Define the Common Identity

The distributed authentication mesh needs a single source of truth. It is not possible to recover user information out of arbitrary information. As an example, an application that uses multiple services with OIDC and Basic Auth needs a common "base of users". Even if the authentication mesh is not in place, the services need to know which basic authentication credentials they need to use for a specific user.

![Definition of the Common Identity](diagrams/04_translator_identity_definition.puml){#fig:04_translator_identity_definition}

As shown in {@fig:04_translator_identity_definition}, the definition of the common identity is quite simple. The only field that needs to be transmitted is the `subject` of a user. The `subject` (or `sub`) field is defined in the official public claims of a JWT [@RFC7519, sec. 4.1.2]. Any additional information that is provided may or may not be used. Since the system is designed to work in a heterogeneous landscape, the common denominator is the users' ID. For some destinations, additional information could be helpful, but it is not guaranteed that the information is available at the source.

### Startup a Translator

When a translator is created with the `NewTranslator()` function of the translator package, a struct type is instantiated that provides some utility functions. Upon creation, the new translators contains two web-servers with the configured ingress and egress ports. Those web-servers are configured to listen to gRPC calls from envoy. The servers in the translator are created but not yet started. They are ready to be run.

![Startup Sequence of a Translator](diagrams/04_package_startup_translator.puml){#fig:04_package_startup_translator}

{@fig:04_package_startup_translator} shows the startup sequence for a translator that was created with the provided package. As soon as the translator gets started (with `translator.Start()`), the translator first ensures its own key material. This means, that a privte key for the local certificate is generated if it does not exist. Further, if the local certificate does not exist, a certificate signing request (CSR) is created and sent to the PKI. Upon success, the PKI returns a valid and signed certificate that the translator can use to sign the JWTs. The last preparation step is to fetch the public part of the CA certificate from the PKI to validate incoming JWTs.

When the preparations for the key material are done, two go routines start the web-servers (listeners) for incoming and outgoing request authentication.

```go
go func() {
	logrus.Info("Serving Ingress")
	err := translator.
		ingressServer.
		Serve(*translator.ingressListen)
	if err != nil {
		logrus.
		WithError(err).
		Fatal("Could not serve ingress.")
	}
}()
```

These listeners are now ready to receive gRPC calls from Envoy. Envoy must be configured to send an authentication check for all intercepted incoming and outgoing calls to the respective destination. The translator now awaits an interrupt, terminate or kill signal to gracefully shut down the listeners.

### Provide Endpoints for Interception

On top of the mentioned listeners are two wrapper methods. A developer provides the ingress and egress function to the library, which in turn then encapsulates the functions with the effective gRPC call from Envoy. When creating a translator, the provided egress function just receives the `CheckRequest` and the ingress function receives a `string` for the parsed subject (user ID) and the `CheckRequest` from Envoy as parameters. They return their respective result (`IngressResult` and `EgressResult`) which then decides the fate of the request.

```go
result, err := server.EgressTranslator(req)
if err != nil {
	return nil, err
}

if result.Skip {
	return envoy.CreateNoopOKResponse(), nil
}

if result.UserID == "" {
	return envoy.CreateForbiddenResponse(
		"No UserID given for outbound communication."
	), nil
}

if result.Forbidden != "" {
	return envoy.CreateForbiddenResponse(result.Forbidden),
		nil
}

return envoy.CreateEgressOKResponse(
	server.JWTConfig,
	result.UserID,
	result.HeadersToRemove
)
```

The function above shows the logic for outgoing (egress) communication. The developer provides the `server.EgressTranslator(req)` implementation at the start of the translator. The package then in turn calls this function and handles the result according to the logic above.

```go
wirePactJWT, ok := req.Attributes.
	Request.Http.
	Headers[wirepact.IdentityHeader]
if !ok {
	return envoy.CreateNoopOKResponse(), nil
}

subject, err := wirepact.GetJWTUserSubject(wirePactJWT)
if err != nil {
	return nil, err
}

result, err := server.IngressTranslator(subject, req)
if err != nil {
	return nil, err
}

if result.Skip {
	return envoy.CreateNoopOKResponse(), nil
}

if result.Forbidden != "" {
	return envoy.CreateForbiddenResponse(
		result.Forbidden
	), nil
}

return envoy.CreateIngressOKResponse(
	result.HeadersToAdd,
	append(result.HeadersToRemove, wirepact.IdentityHeader)
), nil
```

On the other hand, incoming (ingress) communication requires an extra step. First, a check ensures that the authentication mesh header is present. If not, the request gets forwarded to the destination without any interruption. If a JWT is present, the JWT is decoded and the subject (user ID) extracted. The next step involves the provided `server.IngressTranslator` from the developer that coded the translator. The last step is similar to egress communication, where the result of the translator function is parsed and executed accordingly.

### Encode the JWT

Since the chosen technology to transport the users' identity is a JSON web token, the package provides a simple way to en-, and decode the JWTs. One thing that must be provided to the JWT is the subject (i.e. the users ID). Since we defined that the only required thing for the identity is the user ID (see {@fig:04_translator_identity_definition}), we encode it in the official specified JWT way and name it "sub" (i.e. "subject").

![Encode and Sign a JWT](diagrams/04_package_create_jwt.puml){#fig:04_package_create_jwt}

The logic in {@fig:04_package_create_jwt} shows what happens to a user ID if a JWT shall be created. The JWT headers (`x5c` and `x5t`) are created from the local certificate chain and the fetched CA certificate from the PKI. Those headers are used by the receiving party to check if the JWT is valid and from a participant of the authentication mesh. Next, the JWT claims are configured (subject, issuer, expiry date and so forth) and the token is signed. The signed token is then injected into the HTTP call with a special `x-wirepact-identity` header.

### Decode the JWT

On the receiving side, the process is reversed.

![Decode JWT and Extract Subject](diagrams/04_package_decode_jwt.puml){#fig:04_package_decode_jwt}

{@fig:04_package_decode_jwt} shows the process for incoming communication. When the translator receives a call from the outside world that contains the mesh HTTP header (`x-wirepact-identity`), then the process runs. First, the encoded header data is parsed as JWT. Next, the translator checks, if the encoded certificate chain (`x5c`) is valid and matches its own CA certificate. Then the attached certificate hash (`x5t`) is checked against the signing certificate of the source (which must be the first certificate in the provided certificate chain). The headers that are checked are defined in the JWT specification [@RFC7515]. If a subject can be extracted, the developers code will be called and in the end, the request is forwarded if everything went fine.

## Implementing an HTTP Basic Translator with a Secure Common Identity

This section describes the usage of the secure common identity mentioned above within a translator. The translator uses HTTP Basic Auth (RFC7617 [@RFC7617]) for the username/password combination. The implementation is hosted on <https://github.com/WirePact/k8s-basic-auth-translator>.

### Validate and Encode Outgoing Credentials

Any application that shall be part of the authentication mesh must either call the injected forward proxy by itself, or it should respect the `HTTP_PROXY` environment variable, as done by Go and other languages/frameworks. Outgoing communication ("egress") is processed by the envoy proxy with the external authentication mechanism [@buehler:DistAuthMesh]. The following results exist:

- Skip: Do not process any headers or elements in the request
- UserID empty: forbidden request
- Forbidden not empty: for some reason, the request is forbidden
- UserID not empty: the request is allowed

![Skipped/Ignored Egress Request](diagrams/04_translator_egress_skip.puml){#fig:04_translator_egress_skip}

{@fig:04_translator_egress_skip} shows the sequence if the request is "skipped". In this case, skipped means that no headers are consumed nor added. The request is just passed to the destination without any interference. This happens if, in the case of Basic Auth, no `HTTP Authorize` header is added to the request or if another authentication scheme is used (OIDC for example). The possibility to skip a request enables front-facing applications to still receive normal requests that do not contain any authentication information. As an example, this can happen when an application periodically calls some service that does not need any credentials. The neutral request must not be rejected or forbidden by that fact.

![Unauthorized Egress Request](diagrams/04_translator_egress_no_id.puml){#fig:04_translator_egress_no_id}

{@fig:04_translator_egress_no_id} depicts the process when the request contains a correct HTTP header, but the provided username/password combination is not found in the "repository" of the translator. So, no common user ID can be found for the given username and therefore, the provided authentication information is not valid.

![Forbidden Egress Request](diagrams/04_translator_egress_forbidden.puml){#fig:04_translator_egress_forbidden}

In {@fig:04_translator_egress_forbidden}, the HTTP header is present, but corrupted. For example, if the username/password combination was encoded in the wrong format. If this happens, the proxy will reject the request and never bother the destination with incorrect authentication information.

![Processed Egress Request](diagrams/04_translator_egress_ok.puml){#fig:04_translator_egress_ok}

If a request contains the correct HTTP header, the data within is valid and a user can be found with the username/password combination, {@fig:04_translator_egress_ok} shows the process of the request. The translator instructs the forward proxy to consume (i.e. remove) the HTTP authorize header and injects a new custom HTTP header `x-wirepact-identity`. The new header contains a signed JWT that contains the user ID as the subject and the certificate chains as well as a hash of the signing certificate in its headers.

### Validate and Decode an Incoming Identity

The transformer also intercepts incoming connections via the external authentication feature of envoy. If a call contains the specified HTTP header (`x-wirepact-identity`) that contains a JWT, the translator tries to validate the information. In general, there exist four different reactions of the translator:

- Skip: if no information is given that relates to the authentication mesh.
- Error: if any error happens during validation.
- Forbidden: if the request is forbidden for any reason.
- OK: if the request is valid and the information about the user could be gathered.

![Skipped/Ingored Ingress Request](diagrams/04_translator_ingress_skip.puml){#fig:04_translator_ingress_skip}

In {@fig:04_translator_ingress_skip}, the process for a neutral request is shown. The request contains no specific information that is relevant for the authentication mesh. Since the translator may not interfere with requests that are not "part of the mesh", the request is skipped. The destination application may handle the request appropriately. As an example, the target application can request the source of the request to sign in. This process allows normal requests to be handled in the mesh. If all requests without mesh information would be blocked, no "normal" request could be sent.

![Errored Ingress Request](diagrams/04_translator_ingress_error.puml){#fig:04_translator_ingress_error}

Error handling in the translator is bound to return forbidden responses. The translator should not throw any errors if possible [@buehler:DistAuthMesh]. But if there are some errors, the translator returns a forbidden request to deny access to the destination as seen in {@fig:04_translator_ingress_error}. If the translator would just skip the request, this could make the system vulnerable against error attacks, where an attacker could force some error to happen in the translator and then reach the destination.

![Forbidden Ingress Request](diagrams/04_translator_ingress_forbidden.puml){#fig:04_translator_ingress_forbidden}

If the incoming request contains an `x-wirepact-identity` HTTP header and the subject of the user could be extracted successfully, the translator searches for a username/password combination in its repository. If no credentials are found, as shown in {@fig:04_translator_ingress_forbidden}, the request is denied. No valid credentials mean that the translator cannot attach valid basic credentials for the target system.

![Successful Ingress Request](diagrams/04_translator_ingress_ok.puml){#fig:04_translator_ingress_ok}

In contrast to the situations above, {@fig:04_translator_ingress_ok} shows the successful request. If the subject could be parsed, validated and there exists a proper username/password combination in the translators' repository, the translator instructs envoy to consume (i.e. remove) the artificial mesh header and attach the basic authentication header for the target system. In this case, the target system receives valid credentials that it can validate despite the fact that the original source may not have used basic authentication.

### Contrast to an OIDC Translator

Since the HTTP Basic translator mentioned above has a common base with other translators, any other authentication/authorization mechanism can be programmed into a translator. As a further example, and to demonstrate the feasibility of the solution, another translator that handles OpenID Connect (OIDC) was created. The translator resides on GitHub in the repository <https://github.com/WirePact/k8s-keycloak-oidc-translator>.

The OIDC translator is specifically implemented to work in conjunction with a "Keycloak"^[<https://www.keycloak.org/>] instance. Keycloak is an open-source identity and access management (IAM) system that provides the necessary configuration interface to easily use OIDC within your system architecture.

The differences of the OIDC translator to the HTTP Basic Translator are as follows:

- Other configuration (`ISSUER`, `CLIENT_ID`, and `CLIENT_SECRET`) needed
- Reacts to `Authorization: Bearer ...` headers
- Fetches user access token via token exchange^[Explained in the documentation of Keycloak: <https://www.keycloak.org/docs/latest/securing_apps/#_token-exchange>]

The basic logic of the translator remains the same. If an outgoing request contains an authorization HTTP header, the header will be consumed. The access token is validated against Keycloak and if it returns a subject (i.e. user-ID) via the introspection endpoint of Keycloak, the ID is encoded in the JWT and then forwarded. On the receiving side, if the specialized custom HTTP header is attached to any request, the translator tries to extract the users' ID from the request. If successful, the translator acquires a valid service-account access token that has the proper permissions on Keycloak to perform a token exchange with impersonation. With the token exchange, the translator is able to create and fetch a valid access token on behalf of the user. The new access token is attached to the HTTP request and forwarded to the destination. As such, the destination can validate the access token and will receive a valid response from Keycloak. This concept can be adapted to other authentication schemes (e.g. LDAP).

## Automate the Authentication Mesh

The basic concept of the distributed authentication mesh allows the usage of the mesh on all possible platforms. Any platform that wants to participate in the mesh must be able to intercept incoming and outgoing traffic and modify HTTP headers [@buehler:DistAuthMesh]. However, in cloud environments such as Kubernetes, software can be added and removed based on manifest files. In {@sec:definitions}, the concept of an Operator shows how the Kubernetes API can be extended to manage complex applications. But an Operator is not bound to "manage applications". During the implementation phase of this project, an Operator was created for the distributed authentication mesh. It allows users of Kubernetes to dynamically add and remove applications to the mesh via Custom Resource Definitions (CRDs).

The open-source code of the Operator is hosted on GitHub in the repository <https://github.com/WirePact/k8s-operator>. The Operator is written in C\# with the help of the operator SDK "KubeOps"^[<https://github.com/buehler/dotnet-operator-sdk>]. "KubeOps" is an SDK that helps with developing Kubernetes Operators. It abstracts certain aspects of Operators, such as the "watcher" logic that needs to be registered within Kubernetes to receive events about certain entities.

### Use-Cases for the Operator

The Operator must support certain use-cases to have value in the system. It shall help developers to attach applications to the Distributed Authentication Mesh in a declarative way.

![Use-Case Diagram for the Operator](diagrams/04_operator_usecases.puml){#fig:04_operator_usecases}

{@fig:04_operator_usecases} shows the four primary use-cases of the Operator. They are described in a brief form below.

**Ensure PKI:** The Operator must ensure that there exists a PKI. There must only exist one in the system, otherwise, some participants and translators would use the wrong certificate authority. This results in the inability to communicate with other participants.

**Reconcile PKI:** The Operator is responsible to create a valid and correctly configured "Deployment", as well as a "Service" for the created PKI. The deployment will run the PKI with the configured container image and the service will allow participants and translators to call the PKI via a system-wide DNS address.

**Validate Participants:** When a user tries to create a "mesh participant", the Operator is responsible to check if it is valid. For example, the Operator needs to validate that there exists a deployment target and a service that actually want to participate in the authentication mesh. If one of those vital elements do not exist, the Operator shall reject the participant definition.

**Reconcile Participants:** The Operator reconciles "mesh participants". Thus, when a participant is created, the Operator modifies the target deployment and injects the required sidecars and modifies service ports to route the communication through the injected proxy application. Furthermore, the Operator must configure the proxy correctly.

### Custom Entities for Kubernetes

The Operator uses CRDs to manage and reconcile the participants of the Distributed Authentication Mesh. This section describes the custom entities and their specifications in detail.

#### Credential Translator

![Custom Resource Definition for a Credential Translator](diagrams/04_operator_entities_translator.puml){#fig:04_operator_entities_translator}

The credential translator, as shown in {@fig:04_operator_entities_translator}, is one of the core elements in the authentication mesh and the automation engine. This CRD defines translators that the Operator and a mesh participant may use. These definitions can be seen as the "inventory" of the Operator that contains the effective container images for translators. This enables developers to create custom translators and inject them into the mesh even if the core system does not support the particular authentication translator.

```yaml
apiVersion: wirepact.ch/v1alpha1
kind: CredentialTranslator
metadata:
  name: basic-auth
spec:
  image: ghcr.io/wirepact/k8s-basic-auth-translator:latest
```

The declaration above shows an example of such a translator. This is the entity that is stored within Kubernetes when the operator is installed. It does enable the Operator to use the HTTP Basic translator mentioned above. Additionally, the Keycloak OIDC translator is available as well.

#### PKI

![Custom Resource Definition for a PKI](diagrams/04_operator_entities_pki.puml){#fig:04_operator_entities_pki}

{@fig:04_operator_entities_pki} shows the definition for a PKI. When the operator fires up, it checks if a PKI already exists. If not, the operator shall create a PKI such that at most one PKI exists for the mesh. The specification contains the container image, a port, and a (Kubernetes-)secret-name. The port defines on which port the PKI will be available for `/ca` and `/csr` calls and the secret name is a reference to a Kubernetes secret. The secret is used to store the serialnumber, ca certificate, and private key for the PKI. The status of the entity shall be updated by the Operator when a PKI is deployed to the cluster. It must contain the DNS address on which the PKI will be reachable.

#### Mesh Participant

![Custom Resource Definition for a Mesh Participant](diagrams/04_operator_entities_participant.puml){#fig:04_operator_entities_participant}

The mesh participant in {@fig:04_operator_entities_participant} enables developers to actually participate in the distributed authentication mesh by defining a deployment and a service. The specification contains the reference to the targeted Kubernetes deployment as well as the Kubernetes service. The target port enables the Operator to correctly configure the Envoy proxy and adjusting the service. The two properties for `Env` and `Args` enable the Operator to setup the translator that is injected as a sidecar into the deployment. It may contain additional environment variables and/or command-line arguments that are attached to the translator. This could be used to configure a translator that needs special information about a user repository or something similar.

```yaml
apiVersion: wirepact.ch/v1alpha1
kind: MeshParticipant
metadata:
  name: participant
spec:
  deployment: deploy
  service: svc
  targetPort: 8080
  translator: keycloak-oidc-translator
  env:
    ISSUER: http://keycloak.localhost/
    CLIENT_ID: demo
    CLIENT_SECRET: very_secret
```

The example participant above shows the specification needed to run the Keycloak OIDC translator for a deployment ("`deploy`") with a specific service ("`svc`"). The communication on port "`8080`" shall be intercepted by the mesh and the translator in question is "`keycloak-oidc-translator`", for which a `CredentialTranslator` definition with that exact name must exist. As additional environment variables, the issuer, client ID, and client secret variables are passed to the translator such that it can obtain the access tokens from Keycloak.

### Managing the Public Key Infrastructure

One of the use-cases mentioned in {@fig:04_operator_usecases} is managing and reconciling a centralized PKI for the authentication mesh.

![Task for the Startup of the Operator to Ensure a PKI](diagrams/04_operator_pki_startup.puml){#fig:04_operator_pki_startup}

The startup task in {@fig:04_operator_pki_startup} is fairly simple. When the Operator starts, a hosted background service starts (`IHostedService` implementation in .NET) and checks if there are any PKI available. If there are, nothing happens. If not, the Operator creates a PKI entity and stores it within Kubernetes. This will trigger a "normal reconciliation" loop for the entity. This startup process can be further improved by checking the PKI count constantly with a timer instead of just checking when the operator starts. Additionally, a Kubernetes label could ensure that exactly one PKI exists for this particular authentication mesh.

![Reconciliation of a PKI](diagrams/04_operator_pki_reconcile.puml){#fig:04_operator_pki_reconcile}

When a reconciliation loop fires for a PKI definition, the Operator takes several steps to deploy the PKI within its own namespace. {@fig:04_operator_pki_reconcile} shows the steps that are needed to reconcile a PKI. First, the Operator checks if the secret (defined in "`secretName`") exists, then if the deployment and the service exist. If any of those elements does not exist, the Operator creates the entities and stores them within Kubernetes. This reconciliation loop is not very complex and only creates a valid deployment as well as a service to enable access to the PKI within Kubernetes.

### Reconciling Authentication Mesh Participants

The process to reconcile a mesh participant is more complex than the reconciliation of a PKI. To fulfil the use-case "Reconcile Participants" in {@fig:04_operator_usecases}, the operator needs to adjust the specified deployment and service very heavily. When a reconciliation for a `MeshParticipant` is requested, the Operator performs the following actions one by one:

1. Check if the specified translator (`CredentialTranslator` entity) exists in Kubernetes; if not, throw an error.
2. Check if the specified deployment (only `V1Deployment` at the time of writing, without `StatefulSet` and other deployment types) exists in the namespace of the participant; if not, throw an error.
3. Check and update (if needed) the referenced deployment such that it contains the translator and the envoy proxy as a sidecar.
4. Check and update (if needed) the referenced service such that the correct port of the proxy is used instead of the original one.

It is good practice^[According to Kubernetes and several operator SDKs like "KubeOps"] that an Operator reconciliation loop does not differentiate if an entity was just created or updated within Kubernetes. The reconciliation loop shall check if the desired state still matches the actual state in the system. As such, if the entity gets updated or the Operator is requested to reconcile a participant again by any means, the current objects are checked if they are still valid. As a result, the reconciliation of mesh participants is a complex process. The following two sections show a breakdown of the reconciliation loop for mesh participants.

#### Reconcile the Target Deployment

![Reconcile Mesh Participant - Deployment - Preparation](diagrams/04_operator_participant_reconcile_deployment_prepare.puml){#fig:04_operator_participant_reconcile_deployment_prepare}

When the Operator is required to reconcile a mesh participant, several preparation steps are taken. {@fig:04_operator_participant_reconcile_deployment_prepare} shows the first few actions that reconcile the targeted deployment of a participant. If the referenced deployment does not exist in the given namespace, an error is thrown. Further, if the participant does not contain defined ports (for incoming, outgoing, transformer-incoming, and transformer-outgoing communication), they are created and stored. Otherwise, the ports are returned. The last step in preparation is fetching the DNS address for the PKI.

![Reconcile Mesh Participant - Deployment - Translator Container](diagrams/04_operator_participant_reconcile_deployment_translator_container.puml){#fig:04_operator_participant_reconcile_deployment_translator_container}

When the Operator finishes the preparation in {@fig:04_operator_participant_reconcile_deployment_prepare}, the process in {@fig:04_operator_participant_reconcile_deployment_translator_container} takes place. The Kubernetes deployment is analysed more thoroughly. The first check targets the sidecar for the credential translator. If the container definition does not exist, it is created and the environment variables and ports are configured. Then it is attached to the deployment. If the container did exist, it is validated and checked that all required values are as they should be. This step mitigates the risk that the manifest are editted externally since the Operator will reset any changes to the sidecars.

![Reconcile Mesh Participant - Deployment - Envoy Configuration](diagrams/04_operator_participant_reconcile_deployment_envoy_config.puml){#fig:04_operator_participant_reconcile_deployment_envoy_config}

The next step, as {@fig:04_operator_participant_reconcile_deployment_envoy_config} depicts, is validating the proxy (Envoy) configuration. The config is stored in a `V1ConfigMap` in Kubernetes. The config object contains two properties ("`envoy-config.yaml`" and "`config-hash`") that contain the config and a SHA256 hash of the config. If the config object does not exist, the config is created and stored in the config object with the hash. If it does exist, the hash within the object is checked against the config that **should** be stored. When the hash matches, nothing happens. Otherwise, the generated config is stored along with its hash. After the config is validated, the deployment is searched if the associated volume to bind the config as files exists. If not, it is attached to the deployment.

![Reconcile Mesh Participant - Deployment - Envoy Container](diagrams/04_operator_participant_reconcile_deployment_envoy_container.puml){#fig:04_operator_participant_reconcile_deployment_envoy_container}

Second to last step is checking the Envoy container. {@fig:04_operator_participant_reconcile_deployment_envoy_container} shows these steps. The same technique that checks the translator container in {@fig:04_operator_participant_reconcile_deployment_translator_container} ensures the existance and correctness of the Envoy container. The container contains several environment variables and open ports. Further, the container is injected into the deployment when it does not exist.

![Reconcile Mesh Participant - Deployment - Proxy Environment Variable](diagrams/04_operator_participant_reconcile_deployment_proxy_check.puml){#fig:04_operator_participant_reconcile_deployment_proxy_check}

{@fig:04_operator_participant_reconcile_deployment_proxy_check} shows the last step to reconcile the targeted deployment for a mesh participant. All other containers in the deployment receive an environment variable with the local HTTP proxy. The variable `HTTP_PROXY` contains the value "`http://localhost:{egressPort}`". Since multiple containers in a pod run on the same "machine", they share the same localhost. This enables the Distributed Authentication Mesh to configure the application to use a local running Envoy instance as HTTP proxy. Hence, the application routes its outgoing communication through Envoy which then in turn can communicate with the credential translator.

#### Reconcile the Target Service

In contrast to the target deployment reconciliation explained in the section above, the process to reconcile the targeted service is not as complex.

![Reconcile Mesh Participant - Service](diagrams/04_operator_participant_reconcile_service.puml){#fig:04_operator_participant_reconcile_service}

{@fig:04_operator_participant_reconcile_service} shows the steps to reconcile a target service for a mesh participant. If the service does not exist, or it does not contain the target port that is specified in the participant entity, an error is thrown. The target port for this "external service port" is then changed to "`ingress`". The referenced deployment contains this "ingress" reference as external application port for incoming communication on the Envoy sidecar. As such, all incoming communication on the target port is routed to Envoy. Thus, Envoy can intercept the communication and can consult with the credential translator.
