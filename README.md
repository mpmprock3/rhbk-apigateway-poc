# RHBK API Gateway POC

This repository contains Proof of Concept (POC) implementations for securing APIs using **Red Hat Build of Keycloak (RHBK)** with different API Gateway solutions.

## POCs

| POC | API Gateway | Description |
|-----|-------------|-------------|
| [AWS API Gateway](./rhbk-apigateway-poc/) | AWS API Gateway (HTTP API) | Secure an AWS-hosted API using JWT Authorizer with RHBK on OpenShift |
| [Red Hat Connectivity Link](./rhbk-rhcl-poc/) | Kuadrant + Istio (OpenShift Service Mesh 3) | Secure a Kubernetes Gateway API using Kuadrant AuthPolicy with RHBK on OpenShift |

Both POCs use the **OAuth 2.0 Client Credentials Grant** (machine-to-machine) flow with RHBK as the identity provider.
