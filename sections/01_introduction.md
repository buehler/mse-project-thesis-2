\newpage

# Introduction {#sec:introduction}

With the introduction of the concept "Distributed Authentication Mesh" [@buehler:DistAuthMesh], a theoretical base for dynamic authorization in heterogeneous systems was created. The project, in conjunction with the Proof of Concept (PoC) showed, that it is generally possible to transform the identity of a user such that the user can be authorized in another application. In contrast to SAML (Security Assertion Markup Language), the authentication mesh does not require all participants to understand the same authentication and authorization mechanism. The mesh is designed to work within a heterogeneous authentication landscape.

The PoC was designed to show the ability of heterogeneous authentication. The project "Distributed Authentication Mesh" mentioned a "common language format" for the transport, but did not define nor implement it [@buehler:DistAuthMesh]. This project enhances the concept of the "Distributed Authentication Mesh" by evaluating and specifying the transport protocol for the common language between services. The common language is crucial for the success of the mesh. Additionally, an implementation of the mesh for Kubernetes^[<https://kubernetes.io/>] is created during this project. The implementation is tied to Kubernetes itself, but the concept of the mesh can be adapted to various platforms.

The remainder of the report describes prerequisites, used technologies with their terminology and further concepts. The state of the authentication mesh shows the current version of the concept and which elements are missing. The implementation shows the concrete evaluation of the common language format combined with the definition and implementation of the chosen format. Since the implemented version of the mesh runs on Kubernetes, an Operator is created during the implementation to automate the usage of the authentication mesh to allow a good developer experience.