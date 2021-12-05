\newpage

# Implementing a Common Language and Conditional Access

This section analyzes different approaches to create a common language format between the service of the "Distributed Authentication Mesh". After the analysis, the definition and implementation of the common format enhances the general concept of the Mesh and enables a production-grade software.

## Goals and Non-Goals of the Project

This section provides two tables with functional and non-functional requirements for the project. The mentioned tables extend the requirements of the distributed authentication mesh [@buehler:DistAuthMesh, seq. 4.2]. As mentioned, this project enhances the concept of the distributed authentication mesh by analyzing various ways of transmitting the user identity and defining a meaningful way to transport the identity between participants.

tbl.

## A Way to Communicate with Integrity

To enable the translators in the distributed authentication mesh to communicate securely, a common format must be used. The format must support a feasible way to prevent modification of the data it transports. The following sections give an overview over the three options that may be used. In the end of the section a comparison shows pro and contra to each option and a decision is made.

### YAML, XML, JSON, and Others

YAML (YAML Ain't Markup Language^[<https://yaml.org/>]), XML (Extensible Markup Language^[<https://www.w3.org/XML/>]), JSON (JavaScript Object Notation^[<https://www.json.org/>]) and other structured data formats such as binary serialized objects in C\# are widely used for data transport. They are typically used to transport structured data in a more or less human-readable form but maintain the possibility to be serialized and deserialized by a machine. Those structures could be used to transport the identity of an authenticated user. However, the formats do not support integrity checks by default.

```yaml
userId: 123456
userName: Test User
```

The example above shows a simple YAML example with an "object" that contains two properties: `userId` and `userName`. These objects can be extended and well typed in programming languages.

There exist (TODO REF: https://ieeexplore.ieee.org/abstract/document/7856724) approaches like "SecJSON" that enhance JSON with security and integrity mechanisms. But if the standard of the specifications is used, no integrity check can be perfromed and the translators of the authentication mesh cannot validate if the data was not modified. Thus, using a "simple" structured dataformat for the transmission of the user identity would not suffice the security requirements of the system.

TODO: Search other secure methods for xml/yaml
TODO: describe a possible way to implement integrity hashing?

### X509 Certificates

The x509 standard defines how certificates shall be used. Today, the connection over HTTPS is done via TLS and certificate encryption. (TODO CHECK!) The data in a certificate is not fixed, however. There can be "private extensions" (TODO REF to paper/spec) that can be used to transmit data to a possible receiver.

Certificates have the big advantage that they can be integrity checked via already implemented hashing mechanisms and provide a "trust anchor"^[A trust anchor is a root for all trust in the system.] in the form of a root certificate authority (root CA).

But, implementing custom private fields and manipulating that data is cumbersome in various programming languages. In C\# for example, the code to create a simple x509 certificate can span several hundred lines of code. Go^[<https://go.dev/>] on the other hand, has a much better support for manipulating x509 certificates. Since the result of this project should have a good developer experience, using x509 certificates is not be the best solution to solve the communication and integrity issue.

TODO: example of a certificate
TODO: example of private fields
TODO: ref for private x509 fields

### JSON Web Tokens

A plausible way to encode and protect data is to encode them into a JSON web token (JWT). JWTs are used to encode the user identity in OpenID Connect and OAuth 2.0 (TODO REF). A JWT contains three parts (TODO REF): A "header", a "payload" and the "signature". The header identifies which algorithm was used to sign the JWT and can contain other arbitrary information. The payload carries the data that shall be transmitted. The last part of the JWT contains the constructred signature of the header and the payload. This signature is constructed by either a symmetrical or asymmetrical hashing algorithm. (TODO REF, HMAC256 or RSA256).

In OpenID Connect (OIDC) (TODO REF), "Bearer" tokens are used to authenticate a user against a system. The tokens may be opaque or transparent (TODO REF).

- x5c / x5t
  TODO: ref JOSE
  TODO: ref JWT / JWS / JWK

## Implementing a Secure Common Identity

- implement the PKI
- usage of PKI key material
- use key material to sign JWT tokens
- "translator" can validate JWT tokens with key material

## Intercept the traffic

differentiate between service mesh (service discovery) and HTTPPROXY
