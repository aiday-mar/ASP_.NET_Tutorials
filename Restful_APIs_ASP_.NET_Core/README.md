# Building and Securing RESTful APIs in ASP. NET Core

Rest represents 'representational state transfer'. REST is about modelling the API around resources which can be documents, users, orders etc, and allowing clients to perform operations on those resources. The endpoints in a RESTful API represent resources, or collections of resources. There is another type called RPC or Remote Procedural Call.

In REST the endpoints are nouns (resources), and represent sources you can act on. For example you could picture this in terms of links as : "api.com/users" In contrast, endpoints in an RPC API are verbs (methods), which can be represented as : "api.com/getUserInfo". These are methods which can be run on the server.

In a REST API, calling an endpoint acts on a resource using an HTTP method as the verb. An RPC API uses HTTP methods too in the link. Calling an RPC API is like calling a function. Both REST and RPC use HTTP methods. Use REST when the app is data driven. RPC APIs are good when you need to design a strict set of way the client can interact with the server.

REST APIs are also distinguished by the fact that there is the HATEOAS or the Hypermedia feature which acts as the engine of application state. The idea of HATEOAS is that the responses from the API tell the client what it can do in the API. In other words, the responses should include links to the actions available. There should be no need for documentation.

Among  HTTP method, GET and HEAD are read-only and safe.

Idempotent Methods, the effect of mutiple identical requests is the same as one request. Use PUT to create or fully update resources with a known ID. A method is not idempotent whe the resource state on the server changes between tries.

Generally POST is used to create , PUT for full updates, PATCH for partial updates. JSON has eclipsed XML in popularity because it is lightweight and human readable. However it does not have a document schema. HAL is designed for hypermedia, so is ION. In ION links between resources are modelled as JSON objects. Links in ION are specifies as follows :

```
{
  "href" : {link}
  "method" : {GET, POST etc}
  "rel" : [ "self" ]
}
```

A resources is defined as follows where var is a random key, and it has a value val associated to it :

```
{
  "href" : {link}
  "var": "val"
  ...
}
```
ION defines a value object. The ConfigureServices function in the Startup.cs file is where we include depencies used throughout the project. The Configure method sets up the application pipeline. This is where you add the middleware you want to respond to incoming requests. Our ConfigureServices function becomes :

```
public void ConfigureServices(IServiceCollection services){
  
  services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_{version});
  
  services.AddRouting(options => options.LowercaseUrls = true)
}
```
