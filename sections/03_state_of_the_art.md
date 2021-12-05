\newpage

# State of the Authentication Mesh and the Deficiencies {#sec:state_of_the_art}

This section shows the deficiencies that this project tries to solve. Since this project enhances the concepts of the "Distributed Authentication Mesh", many elements are already defined in the past work.

## Common Language Format for Communication

The "Distributed Authentication Mesh" defines an architecture that enables a dynamic conversion of user identities in a declarative way [@buehler:DistAuthMesh]. The common language format however, is neither defined nor implemented yet. To enable the possibility of a production-grade software based on the concepts of the authentication mesh, this part must be specified.

![General communication flow of two services in the distributed authentication mesh](images/03_scope_common_language_format.png){#fig:03_scope_common_language_format short-caption="Authentication Mesh Communication Flow"}

{@fig:03_scope_common_language_format} shows the communication between two services that are part of the distributed authentication mesh. The communication of the app in service A is proxied and forwarded to the translator. The translator then determines if the request contains any relevant authentication information. Then the translator converts the information into a common language format that the translator of service B understands. After this step, the proxy forwards the communication to service B. The proxy in service B will recognize the custom language format in the HTTP headers and uses its transformer to create valid credentials (such as username/password) out of the custom language format, such that the app of service B can authenticate the user. If service A and B use the same authentication scheme, this transformation is not needed. However, if a heterogeneous authentication landscape is present, where service A uses OIDC and service B is a legacy application that only supports Basic Auth, the need for transformation arises.

This common language format is not specified. This project analyzes various forms of such a common language and specifies the language along with the requirements. Furthermore, an implementation shall be provided for Kubernetes to see the concepts in a productive environment.

## Restricting Access to Services with Rules

In the concept of the authentication mesh, a proxy intercepts the communication from and to applications that are part of the mesh. The interception does not interfere with the data stream. The goal of the proxy is the de-, and encoding of the user identity that is transmitted [@buehler:DistAuthMesh]. To provide additional use for this system, a rule based access engine could enhance the usefulness of the distributed authentication mesh.

An additional mechanism ("Rule Engine") could be added to the mesh. This engine takes the configuration from the store and takes place before the translator checks the transmitted identity. A partial list of features could be:

- Timed access: define times when the access to the service is rejected or explicitly allowed.
- IP range: Define IP ranges that are allowed or blocked. This could prevent cross-datacenter-access.
- Custom logic: With the power of a small scripting language^[For example [Lua](https://lua.org) or JavaScript with their respective execution environment], custom logic could be built to allow or reject access to services.

The rule engine should be extensible such that additional mechanisms can be included into it. There are other useful filters that help development teams all over the world to create more secure software.
