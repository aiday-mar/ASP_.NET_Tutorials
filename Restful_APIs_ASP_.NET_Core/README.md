# Building and Securing RESTful APIs in ASP. NET Core

Rest represents 'representational state transfer'. REST is about modelling the API around resources which can be documents, users, orders etc, and allowing clients to perform operations on those resources. The endpoints in a RESTful API represent resources, or collections of resources. There is another type called RPC or Remote Procedural Call.

In REST the endpoints are nouns (resources), and represent sources you can act on. For example you could picture this in terms of links as : "api.com/users" In contrast, endpoints in an RPC API are verbs (methods), which can be represented as : "api.com/getUserInfo". These are methods which can be run on the server.

In a REST API, calling an endpoint acts on a resource using an HTTP method as the verb. An RPC API uses HTTP methods too in the link. Calling an RPC PI is like calling a function. Both REST and RPC use HTTP methods. Use REST when the app is data driven. RPC APIs are good when you need to design a strict set of way the client can interact with the server.

REST APIs are also distinguished by the fact that there is the HATEOAS or the Hypermedia feature which acts as the engine of application state. The idea of HATEOAS is that the responses from the API tell the client what it can do in the API. In other words, the responses should include links to the actions available. There should be no need for documentation.


