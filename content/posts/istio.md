---
title: "Istio - Introduction"
date: 2019-05-04T10:55:10+02:00
draft: false
---
Adoption of microservices architecture is on rising. Teams reap benefits of microservices which enables them to deliver faster, chose a language stack for their choice, quick feedback cycle, etc but, this freedom and pace comes at the cost of complexity to manage a new distributed system which must work in collaboration with each other to deliver the business functionalities. Reliability, fault tolerance, Tracing , Observability are some must-have aspect in a microservice architecture, add to it the functionality of canary deployment, traffic shifting and suddenly there is a whole layer of support system that is required for a service to deliver reliably.
Service Mesh Technologies is giving options to tackle this scenario. Let's see what is a Service Mesh .




A service mesh is a network of microservices and interaction among them. As the network of microservices increases, managing the mesh becomes harder for teams. A service mesh often requires load balancing,metric collection, traffic management, canary rollout,security and many more operations.
Service Mesh technologies manage various aspects of this mesh.Istio is one of Service Mesh technology. 




But before going into the overview of Istio let's think how we used to manage this now without service mesh. For the functionalities that I mentioned above, there are very good libraries available for example in JVM framework we have Netflix hystrix for fault tolerance and circuit breaking, Spring cloud sleuth for distributed tracing, we can use Zipkin and Grafana for visualizing metrics.All these libraries are good and they have an active development going on(Netflix has declared hystrix in maintenance only mode). But this way we are mixing infrastructure logic with our application logic. Moreover, if we want to use a different language we have to use libraries for that language. Service Mesh detaches these responsibilities from the application layer to the infrastructure layer thus these functionalities can be injected in deployment rather than in code.




Let's see an overview of Istio Service Mesh
 
Istio is a service mesh that offers

 * Load balancing
 * Traffic Management 
 * Canary based deployments
 * Fault tolerance 
 * Automatic metric log and traces
 * Security in service to service interaction
 


Istio architecture

![Example image](https://istio.io/docs/concepts/what-is-istio/arch.svg)

Istio has two part

 * Data Plane - Envoy proxy
 * Control Plane - It consists of components named Pilot, Mixer, Citadel, Galley
 
 


Data Plane- 
 
 Istio data plane is made up with Envoy proxy. The proxy is deployed alongside the application in  VM, Container, or POD.Envoy intercepts the traffic and provides traffic management, metric log, fault tolerance capabilities to the mesh.
 
Control Plane-

 * Pilot - Pilot provides traffic management, service discovery for Envoy proxies and resiliency.
 * Mixer - It asserts access control and collect telemetry data across the mesh.
 * Citadel- It provides service to service and end-user security.
 * Galley - Galley is configuration validation and processing component for Istio.
 


Deployment for Istio-
 
 * Kubernetes - For Kubernetes we can use Helm CRDs to install the Istio components. To inject the sidecar envoy proxies there is one automated way where we label the namespace with "istio-injection=enabled". In case of namespace without a label we can use "istioctl kube-inject" to inject envoy proxy to the namespace manually. See  https://istio.io/docs/setup/kubernetes/ for details.
 * Container+Consul - For consul environment with containers we can use DockerCompose files to deploy istio components.We also need Registrator to register the deployed applications to the consul. See  https://istio.io/docs/setup/consul/ for details.
 Istio documentation says it is agnostic of deployment infrastructure but I couldn't find an example of deploying on bare metal servers.
 


Istio eases the job of managing service network and provides support for many operations that the microservice applications need for their resiliency and observability.The development of Istio is very active and continously adding new enhancement and performance betterment into the components. A good start is obviously to go through the bookinfo example in the official documentation. There are also a lot of youtube videos available, one of them  https://www.youtube.com/watch?v=6BYq6hNhceI also have a nice voice command integration in the demo which is nice.
 


Overall Istio service mesh technology is worth watch out for its adoption by organizations in future. Check out  https://istio.io for reference .

