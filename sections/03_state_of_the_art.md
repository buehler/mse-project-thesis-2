\newpage

# State of the Authentication Mesh {#sec:state_of_the_art}

This section shows the deficiencies that this project tries to solve. Since this project enhances the concepts of the "Distributed Authentication Mesh", many elements are already defined in the past work.

## Common Language Format for Communication {.unlisted .unnumbered}

The "Distributed Authentication Mesh" defines an architecture that enables dynamic conversion of user identities in a declarative way [@buehler:DistAuthMesh]. The common language format however, is neither defined nor implemented yet. Past work did implement a Proof of Concept (PoC) to show the general idea, but did not prove the feasibility with the common language format. To enable the creation of a production-grade software based on the concepts of the authentication mesh, the common language must be defined and specified.

![General communication flow of two services in the distributed authentication mesh](images/03_scope_common_language_format.png){#fig:03_scope_common_language_format short-caption="Authentication Mesh Communication Flow"}

{@fig:03_scope_common_language_format} shows the communication between two services that are part of the distributed authentication mesh. The communication of the app in service A is proxied and forwarded to the translator. The translator then determines if the request contains any relevant authentication information. Then the translator converts the information into a common language format that the translator of service B understands. After this step, the proxy forwards the communication to service B. The proxy in service B will recognize the custom language format in the HTTP headers and uses its transformer to create valid credentials (such as username/password) out of the custom language format, such that the app of service B can authenticate the user. If service A and B use the same authentication scheme, this transformation is not needed. However, if a heterogeneous authentication landscape is present, where service A uses OIDC and service B is a legacy application that only supports Basic Auth, the need for transformation arises.

The mentioned common language format is not specified. This project analyzes various forms of such a common language and specifies the language along with the requirements. Furthermore, an implementation shall be provided for Kubernetes to see the concepts in a productive environment.
