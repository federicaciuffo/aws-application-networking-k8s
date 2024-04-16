# AWS Gateway API Controller User Guide

The AWS Gateway API Controller integrates [VPC Lattice](https://aws.amazon.com/vpc/lattice/) with the [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/). When installed in your cluster, the controller watches for the creation of Gateway API objects such as Gateways and Routes and provisions corresponding Amazon VPC Lattice resources. This enables users to configure advanced networking scenarios using Kubernetes APIs, without needing to write custom code or manage sidecar proxies. The AWS Gateway API Controller is an open-source project and fully supported by Amazon.

AWS Gateway API Controller integrates with Amazon VPC Lattice and allows you to:

* Handle network connectivity seamlessly between services across VPCs and accounts.
* Discover these services spanning multiple Kubernetes clusters
* Implement a defense-in-depth strategy to secure communication between those services.
* Observe the request/response traffic across the services.

This documentation describes how to set up the AWS Gateway API Controller, provides example use cases, development concepts, and API references.

While separating the application developer from the details of the underling infrastructure, the controller also provides a Kubernetes-native experience, rather than creating a lot of new AWS ways of managing services.
It does this by integrating with the Kubernetes Gateway API.
This lets you work with Kubernetes service-related resources using Kubernetes APIs and custom resource definitions (CRDs) defined by the Kubernetes [networking.k8s.io specification](https://gateway-api.sigs.k8s.io/references/spec/).

For more information on this technology, see [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/). 
