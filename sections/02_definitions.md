\newpage

# Definitions and Clarification of the Scope {#sec:definitions}

This section provides general information about the project, the context, and prerequisite knowledge. The scope shows what parts of the additional concepts should be considered. Additionally, this section describes the used technologies (Kubernetes and some specific patterns) for this project and a general overview about secure communication between services.

## Scope of the Project

While the project "Distributed Authentication Mesh" addressed the problem of declarative conversion of user credentials (like an access token from an identity provider) [@buehler:DistAuthMesh], this project focuses on the "common language format" that is mentioned in the former project. This project analyses various patterns^[Such as XML, JSON, JWT and so forth.] for such a common language and further implements the common language in Kubernetes. This project provides an analysis of various methods to specify and implement such a common language and gives an implementation for the selected common format.

TODO: sure?
Additionally, it extends the concept of the authentication mesh with a rule engine that allows conditional access control to the destinations of the mesh. As an example, one could configure specific IP ranges that are blocked, specific times when access is allowed or completely custom logic with the help of a scripting language^[Such as Lua: <https://www.lua.org/>].

As for the implementation of the mesh, this project provides an open-source implementation for the public key infrastructure (PKI) that acts as the trust anchor^[Trust Anchor: root source of trust for a system, such as a "root certificate" in certificate chains.] for the mesh. Furthermore, the evaluated pattern for the common language format is implemented in two different "translators" (HTTP Basic Auth and OpenID Connect). An Operator that provides the automation engine of the mesh in context of Kubernetes completes the implementation of the mesh.

Service mesh functions, such as service discovery, are not part of the scope. The authentication mesh should work in conjunction with a service mesh, but does not provide discovery and automated configuration of services. Software that makes use of the authentication mesh must be able to handle the `HTTP_PROXY` and the `HTTPS_PROXY` environment variables to redirect their communication to a forward proxy.

Another topic that is not in the scope of thie project is the authentication and authorization of services against the PKI. While there exist mechanisms to authenticate services, like the usage of a pre-shared key, it is not part of the scope of this project since all elements should reside in the same trust zone. Furthermore, mechanisms such as certificate revocation lists are not implemented in the PKI.

## Kubernetes and its Patterns

This section provides knowledge about Kubernetes and two patterns that are used with Kubernetes. Kubernetes itself manages workloads and load balances them on several nodes (servers) while the patterns enable more complex applications and use-cases.

### Kubernetes, the Orchestrator

Kubernetes is an orchestration software for containerized applications. Originally developed by Google and now supported by the Cloud Native Computing Foundation (CNCF) [@burns:KubernetesBook, ch. 1]. Kubernetes manages the containerized applications and provides access to applications via "Services" that use a DNS naming system. Applications are described in a declarative way in either YAML or JSON.

![The Kubernetes Control Loop](images/02_control_loop.png){#fig:02_control_loop}

{@fig:02_control_loop} shows the Kubernetes "control loop". A controller constantly observes the actual state in the system. When the actual state diverges from the desired one (the one that is "written" in the API in the form of a YAML/JSON declaration) the controller takes action to achieve the desired state. As an example, a deployment has the desired state of two running instances, and currently only one instance is running. The controller will take action and starts another instance such that the actual state matches the desired state.

### An Operator, the Reliability Engineer

The API of Kubernetes is extensible with custom API endpoints (so-called custom resources). With the help of "CustomResourceDefinitions" (CRD), a user can extend the core API of Kubernetes with their own resources [@burns:KubernetesBook, ch. 16]. The custom extensions can be consumed by an Operator to manage complex applications. An Operator runs in Kubernetes and watches for events on CRDs to manage complex applications. Operators can act as controllers for CRDs with the same loop logic shown in {@fig:02_control_loop}.

Operators are like software for Site Reliability Engineering (SRE). The Operator can automatically manage a database cluster or other complex applications that would require an expert with specific knowledge [@dobies:KubernetesOperators].

Two example operators:

- Prometheus Operator^[<https://github.com/prometheus-operator/prometheus-operator>]: Manages instances of Prometheus (open-source monitoring and alerting software).
- Postgres Operator^[<https://github.com/zalando/postgres-operator>]: Manages PostgreSQL clusters in Kubernetes.

A partial list of operators available to use is viewable on [https://operatorhub.io](https://operatorhub.io/).

The Prometheus Operator, for example, introduces several CRDs such as `Prometheus`, `ServiceMonitor` and `Alertmanager` [@web:PrometheusOperatorDocumentation]. When the Operator is installed into Kubernetes, it reacts to create, update and delete events of `Prometheus` resources. Such a resource could be:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: my-custom-prometheus
spec:
  replicas: 2
  serviceAccountName: prometheus
  serviceMonitorSelector:
    matchLabels:
      foo: bar
```

When the resource above is created or updated in Kubernetes, the Operator will be notified by the Kubernetes API. The Operator then creates a `StatefulSet`^[A form of deployment like `V1Deployment` but with certain stateful mechanics inside Kubernetes.] that runs the Prometheus Docker image with the scale of two instances. Also, the application will be configured to use a service account named `prometheus` to run in Kubernetes and will automatically search for `ServiceMonitor` resources with a matching label (`foo: bar`) to scrape^[Scraping: fetch the metrics from the target system and store them with time information].

Operators can be created by any means that interact with the Kubernetes API. Normally, they are created with some SDK that abstracts some of the more complex topics (like watching the resources and reconnection logic). The following non-exhaustive list shows some frameworks that support Operator development:

- kubebuilder^[<https://book.kubebuilder.io/>]: Go^[<https://golang.org/>] Operator Framework
- KubeOps^[<https://buehler.github.io/dotnet-operator-sdk/>]: .NET Operator SDK
- Operator SDK^[<https://operatorframework.io/>]: SDK that supports Go, Ansible^[<https://www.ansible.com/>] or Helm^[<https://helm.sh/>]
- shell-operator^[<https://github.com/flant/shell-operator>]: Operator that supports bash scripts as hooks for reconciling

Operators, and software that implements the Operator pattern, are the most complex extension possibility for Kubernetes, but also the most powerful one [@burns:KubernetesBook]. With Operators, whole applications can be automated in a declarative and self-healing way.

### A Sidecar, the Extension

Sidecars enhance a "Pod"^[The smallest possible workload unit in Kubernetes. A Pod contains of one or more containers that run a containerized application image.] by injecting additional containers to the defined one [@burns:DesignPatternsForContainerSystems].

![A Sidecar example](images/02_sidecar_example.png){#fig:02_sidecar}

{@fig:02_sidecar} shows an example: A containerized application runs in its Docker^[<https://www.docker.com/>] image and writes logs to `/var/logs/app.log` in the shared file system. A specialized "Log Collector" sidecar can be injected into the Pod and read those log messages. Then the sidecar forwards the parsed logs to some logging software like Graylog^[<https://www.graylog.org/>].

## Securing Communication

This section provides the required knowledge about security for this report. Authentication and authorization are big topics in software engineering and there are various standards in the industry. Two of these standards are described below as they are used in this project to show the use-case of the authentication mesh.

### HTTP Basic Authentication

The "Basic" authentication scheme is defined in **RFC7617**. Basic is a trivial authentication scheme which provides an extremely low security when used without HTTPS. It does not use any real form of encryption, nor can any party validate the source of the data. To transmit basic credentials, the username and the password are combined with a colon (`:`) and then encoded with Base64. The encoded result is transmitted via the HTTP header `Authorization` and the prefix `Basic` [@RFC7617].

### OpenID Connect

OIDC (OpenID Connect) is defined by a specification provided by the OpenID Foundation (OIDF). However, OIDC extends OAuth, which in turn is defined by **RFC6749**. OIDC is an authentication scheme that extends `OAuth 2.0`. The OAuth framework only defines the authorization part and how access is granted to data and applications. OAuth, or more specifically the RFC, does not define how the credentials are transmitted [@RFC6749].

OIDC extends OAuth with authentication, that it enables login and profile capabilities. OIDC defines three different authentication flows: `Authorization Code Flow`, `Implicit Flow` and the `Hybrid Flow`. These flows specify how the credentials must be transmitted to a server and in which format they return credentials that can be used to authenticate an identity [@spec:OIDC]. As an example, a user wants to access a protected API via web GUI. The user is forwarded to an external login page and enters the credentials. When they are correct, the user gets redirected to the web application which can fetch an access token for the user. This access token can be transmitted to the API to authenticate and authorize the user. The API is able to verify the token with the login server and can reject or allow the request.

### Trust Zones and Zero Trust

Trust zones are the areas where software "can trust each other". When an application verifies the presented credentials of a user and allows a request, it may access other resources (such as APIs) on the users' behalf. In the same trust zone, other resources can trust the system, that the user has presented valid credentials.

![Example of a Trust Zone](images/02_trust_zones.png){#fig:02_trust_zones}

As an example, we consider {@fig:02_trust_zones}. The API gateway is the only way to enter the trust zone. All applications ("Frontend Application" and "Additional API" among others) are shielded from the outside and access is only granted via the gateway. In this scenario, a user first accesses the frontend application and is redirected to the login page. According to the authorization code flow, the frontend application can fetch an access token when the user returns from the login. Then, the user presents his OIDC credentials via HTTP header to the frontend application and the app can verify the token with the IAM (Identity and Access Management) if the credentials are valid. Since the additional API resides in the same trust zone, it does not need to check if the credentials are valid again, the frontend can call the API on the users' behalf.

In contrast to trust zones, "Zero Trust" is a security model that focuses on protecting (sensitive) data [@iftekhar:ProtectDataWithZeroTrust]. Zero trust assumes that every call could be an intercepted by an attacker. Therefore, all requests must be validated. As a consequence, the frontend in {@fig:02_trust_zones} is required to send the user token along with the request to the API and the API checks the token again for its validity. For the concept of zero trust, it is irrelevant if the application resides in an enterprise network or if it is publicly accessible.

A concern to address, in zero trust or authentication in general, is the authentication of the authenticator. Who assures that, in the given example, the IAM is not a corrupted instance that allows attackers to inject faulty information? Such authentication software is hardened and developed over several months and years. It is not possible to create a perfect safe solution. But a partial solution is to use well-known software^[For example "Auth0", "Keykloak" or "Octa" among others.] and applications to provide the safest possible implementations of such software.
