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

ControllerBase class is a stripped down version without the view and with razor features you only need in a web app. Below we have the starting code for the root contoller.

```
namespace App.Controllers {
  
  [Route("/")]
  [ApiController]   // Api controllers have some extra features
  [ApiVersion("1.0")]
  public class RootController : ControllerBase    // Here it extends the ControllerBase as decribed above
  {
    [HttpGet(Name = nameof(Get))]   // so here we assign to the Name property the name of the following method
    [ProducesResponseType(200)]  // tells asp .net core and swagger ui that this method could return 200, an optional attribute 
    public IActionResult Get()  // this Get method specifies what to do with an incoming HTTP request
    {
      var response = new {
        href = Url.Link(nameof(Get), null)   // no parameters and here we take the name of the Get method hereby
        rooms = new 
        {
          // here we access the GetRooms method from another controller, the one described below
          href = Url.Link(nameof(RoomsController.GetRooms), null)
        }
      };
      
      rerturn Ok(response);
    }
  }  
}
```

We use Postman to make HTTP requests to the API. To use postman you need to copy the sslPort number from the launchSettings.json file, then you make a get request to the following link : `https://localhost:{sslPort}`. Now we create a new controller which will manage rooms as follows :

```
namespace App.Controllers
{
  [Route("/[controller]")]  // here we have that RoomsController is stripped from the Controller and the name is appended to the end of the route
  [ApiController]
  public class RoomsController : ControllerBase{
  
    [HttpGet(Name = nameof(GetRooms))]
    public IActionResult GetRooms() 
    {
      throw new NotImplementedException();
    }
  }
}
```

OpenAPI is a format for describing RESTful APIs. OpenAPI is a system of retrospection/reflection for your API. A file called swagger.json is used as a strongly types client for your API, or it is used to generate API tests. Swagger UI helps to interact with the API in real time. To add OpenAPI to the project you can use either SwashBuckle or NSwag. For example first in the nuget package manager, you can install NSwage.AspNetCore, then you can add the following code to the Configure method.

```
if (env.IsDevelopment()){

  app.UseDeveloperExceptionPage();
  
  app.UseSwaggerUi3WithApiExplorer( options =>
  {  
    options.GeneratorSettings.DefaultPropertyNameHandling =
    NJsonSchema.PropertyNameHandlign.CamelCase;
  });
  
} 
```

Then you go to the localhost with the corresponding ssl port and you find a way to quickly test the API you are building.

There are several ways to version the API. The versioning can happen through the url in the following ways.

```
GET /rooms
Accept : application/ion+json; v = 2.0
```

Or as follows :

```
https://example.io/v1/rooms
```
The later is not a good idea because there is no unique identifier. The first type is more common, it is called the media type (header) versioning approach. Next in the nuget package manager you can install the following `Microsoft.AspNetCore.Versioning`. Then we can add some code to implement the versioning into the ConfigureServices method in the Startup.cs file. We write the following code :

```
services.AddApiVersioning(options => {

  options.DefaultApiVersion = new ApiVersion(1,0);
  options.ApiVersionReader = new MediaTypeApiVersionReader();
  options.AssumeDefaultVersionWhenUnspecified = true;
  options.ReportApiVersions = true;
  options.ApiVersionSelector = new CurrentImplentationApiVersionSelector(options);
});
```
