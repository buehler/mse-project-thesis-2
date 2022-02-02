# Abstract

The "Distributed Authentication Mesh" in [@buehler:DistAuthMesh] is a concept to dynamically convert authentication information (such as access tokens from OpenID Connect) to other authentication schemes (like HTTP Basic). In contrast to "Security Assertion Markup Language" (SAML), the concept does not require all participants to share the same authentication scheme. It eliminates the requirement to introduce code changes into existing applications such that they can support other authentication schemes.

A central part of the mesh is the "common language format". This format is eminently important to the mesh because it delivers the users' identity to other participants. While the previous project included the concept of the mesh and implemented a Proof of Concept for the modification of HTTP headers, it did not provide a definition nor implementation for the common language format.

This project targets the topic of the common language and analyzes several possibilities for such a format. The project also defines the objects that must be transmitted between mesh participants. The concept of the mesh is extended with a "Rule Engine" that improves the security and versatility of the mesh. Additionally, this project implements the "Distributed Authentication Mesh" as open-source software such that it can be operated on Kubernetes. The conclusion provides further information about the project and possible topics of follow-up work.

\newpage

---
