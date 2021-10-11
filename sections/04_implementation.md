\newpage

# Implementing a Common Language and Conditional Access

This section analyzes different approaches to create a common language format between the service of the "Distributed Authentication Mesh". After the analysis, the definition and implementation of the common format enhances the general concept of the Mesh and enables a production-grade software.

## Goals and Non-Goals of the Project

## A Way to Communicate with Integrity

### YAML, XML, JSON, and Others

### X509 Certificates

### JSON Web Tokens

## Implementing a Secure Common Identity

- implement the PKI
- usage of PKI key material
- use key material to sign JWT tokens
- "translator" can validate JWT tokens with key material

## Intercept the traffic

## Limiting Access with a Rule Engine

- Create a format / definition for rules that can limit the access to services
- This is "conditional access"
- Create / implement a rule engine that runs as sidecar (like translator)
