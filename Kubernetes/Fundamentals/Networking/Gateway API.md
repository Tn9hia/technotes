## Resource model
Gateway API has three stable API kinds:

- **GatewayClass**: Defines a set of gateways with common configuration and managed by a controller that implements the class.
- **Gateway**: Defines an instance of traffic handling infrastructure, such as cloud load balancer.
- **HTTPRoute**: Defines HTTP-specific rules for mapping traffic from a Gateway listener to a representation of backend network endpoints. These endpoints are often represented as a Service.

![A figure illustrating the relationships of the three stable Gateway API kinds](https://kubernetes.io/docs/images/gateway-kind-relationships.svg)
### GatewayClass
Gateways can be implemented by different controllers, often with different configurations. A Gateway must reference a GatewayClass that contains the name of the controller that implements the class.

### Gateway
**A Gateway** describes an instance of traffic handling infrastructure. **It defines a network endpoint that can be used for processing traffic, i.e. filtering, balancing, splitting, etc. for backends such as a Service**. For example, a Gateway may represent a cloud load balancer or an in-cluster proxy server that is configured to accept HTTP traffic.

### HTTPRoute
**The HTTPRoute** kind specifies routing **behavior of HTTP requests from a Gateway listener to backend network endpoints**. For a Service backend, an implementation may **represent the backend network endpoint as a Service IP or the backing EndpointSlices of the Service**. An HTTPRoute represents configuration that is applied to the underlying Gateway implementation. For example, defining a new HTTPRoute may result in configuring additional traffic routes in a cloud load balancer or in-cluster proxy server.

## Request flow
![A diagram that provides an example of HTTP traffic being routed to a Service by using a Gateway and an HTTPRoute](https://kubernetes.io/docs/images/gateway-request-flow.svg)

In this example, the request flow for a Gateway implemented as a reverse proxy is:

1. The client starts to prepare an HTTP request for the URL `http://www.example.com`
2. The client's DNS resolver queries for the destination name and learns a mapping to one or more IP addresses associated with the Gateway.
3. The client sends a request to the Gateway IP address; the reverse proxy receives the HTTP request and uses the Host: header to match a configuration that was derived from the Gateway and attached HTTPRoute.
4. Optionally, the reverse proxy can perform request header and/or path matching based on match rules of the HTTPRoute.
5. Optionally, the reverse proxy can modify the request; for example, to add or remove headers, based on filter rules of the HTTPRoute.
6. Lastly, the reverse proxy forwards the request to one or more backends.

##