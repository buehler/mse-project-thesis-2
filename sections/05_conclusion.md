\newpage

# Conclusions and Outlook

This project further improved the core concept of the "Distributed Authentication Mesh" proposed in "Distributed Authentication Mesh" [@buehler:DistAuthMesh]. The goals of this project were the definition and implementation of the "common language format" and a complete implementation of in a cloud environment (i.e. "Kubernetes").

{@sec:introduction} introduces the reader into the general topic of the project and shows references to past work. Further, the past work is briefly analyzed and the goals for this project are outlined.

{@sec:definitions} defines the scope of this project and introduces readers into the technologies used by this project. Kubernetes, the central technology in this project, and some of the required patterns, such as the Operator pattern and the Sidecar pattern, are explained. {@sec:definitions} also gives an overview of the used security mechanisms in this project.

The next section, {@sec:state_of_the_art}, describes the actual state of the "Distributed Authentication Mesh". The concept of the mesh only exists in a theoretical form and the only form of proof that exists is a Proof of Concept (PoC). The PoC does show that it is possible to change HTTP headers for HTTP requests in-flight but it does not show how this may be used for the mesh. Also, the often referenced "common language format" is not defined nor implemented.

The core of this project, the analyzation and implementation of the common language and a practical implementation of the mesh, resides in {@sec:implementation}. Since the concept of the "Distributed Authentication Mesh" only uses the term "common language format", a useful implementation of the system was not possible.

{@sec:implementation} analyzes various data formats, such as structured formats (JSON, YAML, etc.), x509 certificates, and JSON Web Tokens (JWT). Then, a comparison of the formats shows the decision process for the JWT format. Structured formats would be feasible for such a use-case, but they lack the possibility of validating the integrity of the data without implementing further concepts. On the other hand, x509 certificates provide such a mechanism and are already enabled though the existence of a Public Key Infrastructure (PKI) in the authentication mesh. The storage of custom data is possible withing certificates [@RFC5280, sec. 4]. However, they tend to be cumbersome to use in various programming languages. Since one goal of the authentication mesh is a good developer experience, x509 are not the most likely choice. JSON Web Tokens are selected as common language format because of the ease-of-use and the possibility to sign them with an algorithm [@RFC7519]. The JSON Web Signature (JWS) [@RFC7515] in conjunction with a JWT enables data to be transmitted securely.

The reminder of {@sec:implementation} shows the definition and implementation of the PKI, a translator base, an HTTP Basic translator, an OIDC translator, and the Kubernetes Operator that automates the whole authentication mesh. All these sub-sections show (where needed) use-case diagrams and process/sequence/invocation diagrams to explain the implementation further. All created software is available as open-source on GitHub under the organization "WirePact" (<https://github.com/WirePact>). To test the authentication mesh in Kubernetes, the GitHub organization provides a demo application that installs the Operator and a simple application with "Keycloak" and Basic Auth. The code and installation instruction can be found on GitHub (<https://github.com/WirePact/k8s-demo-application>).

The goal for future work is to specify and implement the concept of a "Rule Engine" to further improve the "Distributed Authentication Mesh". To provide additional use for the mesh, a rule based access engine could enhance the usefulness of the distributed authentication mesh. Such a rule engine would further improve the security of the overall system. This engine takes the configuration from the configuration store and takes place before the translator checks the transmitted identity. A partial list of features could be:

- Timed access: define times when the access to the service is rejected or explicitly allowed.
- IP range: Define IP ranges that are allowed or blocked. This could prevent cross-datacenter-access.
- Custom logic: With the power of a small scripting language^[For example [Lua](https://lua.org) or JavaScript with their respective execution environment], custom logic could be built to allow or reject access to services.

The rule engine should be extensible such that additional mechanisms can be included into it. There are other useful filters that help development teams all over the world to create more secure software.

In this project, the state-of-the-art of the concept and the implementation base on the assumption that all participants reside in the same trust context. Extending the concepts and the implementation of this project to enable the "Distributed Authentication Mesh" to be truly "distributed".

With the implementation of the authentication mesh, as documented in {@sec:implementation}, the system can be used in a Kubernetes cloud environment. This enables developers and companies to use legacy applications in conjunction with modern state-of-the-art software without changing one of the mentioned applications. The authentication mesh does dynamically transform user information from the source system into a signed JWT and back to the authentication scheme of the destination.
