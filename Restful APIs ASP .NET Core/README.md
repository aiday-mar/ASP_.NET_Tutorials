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
We write the code as follows :

```
namespace App.Models
{
  public abstract class Resource : Link  // The resource inherits from the Link class defined below
  {
    [JsonIgnore]
    public Link Self {get; set;}
  }
}
```

ION defines a value object. The ConfigureServices function in the Startup.cs file is where we include depencies used throughout the project. The Configure method sets up the application pipeline. This is where you add the middleware you want to respond to incoming requests. Our ConfigureServices function becomes :

```
public void ConfigureServices(IServiceCollection services){
  
  services.AddMvc(options => {
    options.Filters.Add<JsonExceptionFilter>();
    options.Filters.Add<RequireHttpsOrCloseAttribute>();
  }).SetCompatibilityVersion(CompatibilityVersion.Version_{version});
  
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
      var response = new RootResponse
      {
        Self = Link.To(nameof(Get)),
        Rooms = Link.ToCollection(nameof(RoomsController.GetRooms)),
        Info = Link.To(nameof(InfoController.GetInfo)),
      }
      
      rerturn Ok(response);
    }
  }  
}
```

The code for Link.To is specified further down. We use Postman to make HTTP requests to the API. It is a good idea to disable the ssql certificate tempoarily. To use postman you need to copy the sslPort number from the launchSettings.json file, then you make a get request to the following link : `https://localhost:{sslPort}`. Now we create a new controller which will manage rooms as follows :

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

If an unhandled exception occurs in the API we throw an HTTP 500 exception error. Now we can create a model for the API error, the model will be as follows :

```
public class ApiError {
  public string Message {get; set;}
  public string Detail {get; set;}
}
```
Then in a folder called Filters, we create a filter file which we call : `JsonExceptionFilter.cs`. Next we write the following code :

```
namespace App.Filters
{
  public class JsonExceptionFilter : IExceptionFilter
  {
    private readonly IHostingEnvironment _env;
    
    // the below injects the corresponding IHostingEnvironment into the class
    public JsonExceptionFilter(IHostingEnvironment env)
    {
      _env = env;
    }
    
    public void OnException(ExceptionContext context)
    {
      var error = new ApiError();
      
      if(_env.isDevelopment())    // meaning when we are actually developing the application then we have the following error message and the error details
      {
        error.Message = context.Exception.Message;
        error.Detail = context.Exception.StackTrace;
      }
      else 
      {
        error.Message = "A server error occured.";
        error.Details = context.Exception.Message;
      }
      
      context.Result = new ObjectResult(error) 
      {
        StatusCode = 500
      };
    }
  }
}
```

The filter is run before or after ASP .NET core processes a request. The context.Result in a way returns a response from the HTTP request.


<h3><strong>Securing the Api</strong></h3>

There are different types of securities. For example 'transport security' meaning keeping the connection between the client and the server secure. 'Application security' is linked to authentication and the fact that users are allowed to perform certain actions. 

For transport security, some measures that you could take include for example rejecting connection attempts from HTTP and allowing only HTTPS connection attempts. ASP .NET core web applications typically run on the Kestrel web server. Kestrel can be deployed directly or with a reverse proxy like IIS or NGINX.

The HTTP Strict Transport Security Header or the HSTS. The HSTS header is returned by the server and instructs web browesers to prever the user from accidentally connecting over HTTP. The server will refuse to connect unless the header corresponds to HTTPS. ASP .NET core 2.0 and later come with https turned on by default. The Startup.cs file is initiliazed with the following code at the bottom:

```
else 
{
  app.UseHsts();
}
app.UseHttpsRedirection();
app.UseMvc();
```

You can enforce the website to not accept HTTP requests by creating the following file : `RequireHttpsOrCloseAttribute`.

```
namespace App.Filters
{
  public class RequireHttpsOrCloseAttribute : RequireHttpsAttribute{
    protected override void HandleNonHttpsRequest (AuthorizationFilterContext filterContext) 
    {
      filterContext.Result = new StatusCodeResult(400);
    }
  }
}
```

Another security concern when building APIs is dealing with CORS or Cross-Origin Resource Sharing. This is a concern only if the API is accessed by the browser, and it is hosted on a separate domain from the rest of the app. For example this is the case when you host your app on 'example.com' and your get your resources from 'api.example.com'. Modern browsers enforce the Same-Origin Policy. The origin of a page is defined by the combination of the scheme, the host and the port number. But if you want to use different origins, then this is where CORS comes in. CORS allows to define some origins in the whitelist and relax the browser Same-Origin Policy. In the ConfigureService method of Startup.cs we can write :

```
services.AddCors(options =>
{
  options.AddPolicy("AllowMyApp" policy => policy.AllowAnyOrigin())
});

...

app.UseCors("AllowMyApp");
```

<h3><strong>Represent resources </strong></h3>

We make a simple abstract class called Resource which we will extend in order to create other resources.

```
namespace App.Models {
  
  public abstract class Resource 
  {
    [JsonProperty(Order = -2)]
    public string Href {get; set;}
  }
}
```

ASP .NET core is set up to read appsettings.json file from the startup. Suppose you had some json data in a file and it corresponded to the class Info then you could inject this inton the app by writing under the ConfigureServices method the following code :

```
public void ConfigureServices (IServiceCollection services) {
  services.Configure<Info>(Configuration.GetSection("Info"));
  ...
}
```
The above code will return the data as a RESTful resources. We can create a controller to return the above data. Below we are returning a strongly typed model.

```
namespace App.Controllers{
  
  public class InfoController : ControllerBase
  {
    private readonly HotelInfo _hotelInfo;
    
    public InfoController(IOptions<HotelInfo> hotelInfoWrapper)
    {
      _hotelInfo = hotelInfoWrapper.Value;
    }
    
    [HttpGet(Name = nameof(GetInfo))]
    [ProducesResponseType(200)]
    public ActionResult<HotelInfo> GetInfo()
    {
      _hotelInfo.Href = Urk.Link(nameof(GetInfo), null);
      return _hotelInfo;
    }
  }
}
```

You can swap an in-memory database for a real database once you  go into production. You can do this by adding the corresponding code into the ConfigureServices as follows :

```
public void ConfigureServices(IServiceColletion services){
  
  services.AddDBcontext<AppDbContext>(options =>
     options.UseInMemoryDatabase("db")
  );
}
```

And we create a new file for our in memory database :

```
namespace App
{
  public class AppDbContext : DbContext
  {
    public AppDbContext(DbContextOptions options) : base(options) {}
    public DbSet<RoomEntity> Rooms {get; set;}
  }
}
```

Here RoomEntity is a model for the rooms that we are renting out. Since the data is in memory this means that the data is lost when the API restarts. Next we want to seed the database with test data as follows :

```
namespace App {
  public static class SeedData
  {
    public static async Task InitializeAsync(IServiceProvider services)
    {
      await AddTestData(   // you need to wait until this AsyncTask finished
        services.GetRequiredService<AppDbContext>() 
        // the service associated to the context
      );
    }
    
    public static async Task AddTestData(AppDbContext context)
    {
      if (context.Roomy.Any())
      {
        //already has data
        return;
      }
      context.Rooms.Add(new RoomEntity {
        DATA HERE
      });
      
      await context.SaveChangesAsync();
    }
  }
}
```

In program.cs file we write :

```
namespace App
{
  public class Program
  {
    public static void Main(string[] args)
    {
      var host = CreateWebHostBuilder(args).Build();
      InitializeDatabase(host);
      host.Run();
    }
    
    public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
      WebHost.CreateDefaultBuilder(args)
      .UseStartup<Startup>();
      // where startup is the class specified earlier
      
    public static void InitializeDatabase(IWebHost host)
    {
      using (var scope = host.Services.CreateScope())
      {
        var services = scope.ServiceProvider;
        
        try 
        {
          SeedData.InitializeAsync(services).Wait();  // add .Wait() at the end because we don't have a synchronous context
        }
        catch (Exception ex)
        {
          var logger = services.GetRquiredService<ILogger<Program>>();
          logger.LogError(ex, "An  error occured while seeding the database");
        }
      }
    }
  }
}
```
Now we want to return a resource from the controller : 

```
public class RoomsController : ControllerBase 
{
  private readonly IRoomService _roomService;
  private readonly IOpeningService _openingService;
  private readonly PagingOptions _defaultPagingOptions;
  
  public RoomsController(IRoomService roomService, IOpeningService openingServices, IOptions<PagingOptions> defaultPagingOptionsWrapper)
  {
    _roomService = roomService;
    _openingService = openingService;
    _defaultPagingOptions = defaultPagingOptionsWrapper.Value;
  }
  
  [HttpGet(Name = nameof(GetRooms))]
  [ProducesResponseType(200)]
  public async Task<ActionResult<Collection<Room>>> GetRooms()
  {
    var rooms = await _roomService.GetRoomsAsync();
    var collection = new Collection<Room>
    {
      Self = Link.To(nameof(GetRooms)),
      Value = rooms.ToArray();
    };
    
    return collection;
  }
  
  [HttpGet("openings", Name = nameof(GetAllRoomOpenings))]
  [ProducesResponsesType(400)]
  [ProducesResponseType(200)]
  public async Task<ActionResult<Collection<Opening>>> GetAllRoomOpenings(
  [FromQuery] PagingOptions pagingOptions = null)
  {
    pagingOptions.Offset = pagingOptions.Offset ?? _defaultPagingOptions.Offset;
    pagingOptions.Limit = pagingOptions.Limit ?? _defaultPagingOptions.Limit;

    var openings = await _openingService.GetOpeningsAsync(pagingOptions);
    
    var collection = PagedCollection<Opening>.Create(
    Link.ToCollection(nameof(GetAllRoomOpenings)),
    openings.Items.ToArray(),
    openings.TotalSize,
    pagingOptions
    );
    
    return collection;
  }
  
  [HttpGet("{roomId}", Name = nameof(GetRoomsById))]
  [ProducesResponseType(404)]
  [ProducesResponseType(200)]
  public async Task<ActionResult<Room>> GetRoomById(Guid roomId)
  {
    var room = await _roomService.GetRoomAsync(roomId);
    if (room == null) return NotFound();
    return room;
  }
}
```

Where we used the following code for a service :

```
namespace App.Services
{
  public class DefaultRoomService : IRoomService
  {
    private readonly AppDbContext _context;
    private readonly IConfigurationProvider _mappingConfiguration
    
    public DefaultRoomService(AppDbContext context, IConfigurationProvider mappingConfiguration)
    {
      _context = context;
      _mappingConfiguration = mappingConfiguration;
    }
    
    public Task<Room> GetRoomsAsync(Guid id)
    {
      
       var entity = await _context.Rooms.SingleOrDefaultAsync(x => x.Id == id);

        if(entity == null)
        {
          return null;
        }
        
        var mapper = _mappingConfiguration.CreateMapper();
        return mapper.Map<Room>(entity);
    }
    
    public async Task<IEnumerable<Room>> GetRoomsAsync()
    {
      var query = _context.Rooms.ProjectTo<Room>();
      
      return await query.ToArrayAsync();
    }
}
```

In the above we used a mapper which is written below :

```
namespace App.Infrastructure
{
  public class MappingProfile : Profile
  {
    public MappingProfile()
    {
      CreateMap<RoomEntity, Room>().ForMember(dest => dest.Rate, opt => opt.MapFrom(src => src.Rate/100.0m))
      .ForMember(dest => dest.Self, opt => opt.MapFrom(src =>
        Link.To(nameof(Controllers.RoomsController.GetRoomsById), new {roomId = src.Id}));
    }
  }
}
```
We need to add the above service to the configure services method in Startup.cs

```
services.AddAutoMapper(
  options => options.AddProfile<MappingProfile>
);
```
Then after we specify the Link class which is used in the RootController.

```
namespace App.Models
{
  public class Link
  {
    pulic const string  GetMethod = "GET";
    
    public Static Link To(string routeName, object routeValue = null) => new Link
    {
      RouteName = routeName,
      RouteValues = routeValues,
      Method = GetMethod, 
      Relations = null,
    }
    
    public Static Link ToCollection(string routeName, object routeValue = null) => new Link
    {
      RouteName = routeName,
      RouteValues = routeValues,
      Method = GetMethod, 
      Relations = new[] {"collection"},
    }
    
    [JsonProperty(Order = -4)]
    public string Href {get; set;}
    
    // the below code means that you ignore the null values
    [JsonProperty(Order = -3, PropertyName = "rel", NullValueHandling = NullValueHandling.Ignore)]
    public string[] Relations {get; set;}
    
    [JsonProperty(Order = -2, DefaultValueHandling = DefaultValueHandling.Ignore, NullValueHandling = NullValueHandling.Ignore)]
    [DefaultValue(GetMethod)]
    public string Method {get; set;}
    
    [JsonIgnore]
    public string RouteName {get; set;}
    
    [JsonIgnore]
    public object RouteValues {get; set;}
  }
}
```

Among the resources that we can return, it is possible to return collections. The corresponding code is :

```
namespace App.Models
{
  public class Collection<T> : Resource
  {
    public T[] Value {get; set;}
  }
}
```

Below we define the IRoomService which we used in some of the codes above :

```
namespace App.Services
{
  public interface IRoomService
  {
    Task<IEnumerable<Room>> GetRoomsAsync(PagingOptions pagingOptions, 
    SortOptions<Room, RoomEntity> sortOptions);
    Task<Room> GetRoomsAsync(Guid id);
  }
}
```
In a similar manner we have :

```
namespace App.Models
{
  public class PagedCollection<T> : Collection<T>
  {
    public static PagedCollection<T> Create(Link self, T[] items, int size, PagingOptions pagingOptions) 
    => new PagedCollection<T>
    {
      Self = self,
      Value = items,
      Size = size,
      Offset = pagingOptions.Limit,
      First = self,
      Next = GetNextLink(self, size, pagingOptions),
      Previous = GetPreviousLink(self, size, pagingOptions),
      Last = GetLastLink(self, size, pagingOptions),
    }
    
    [JsonProperty(NullValueHandling = NullValueHandling.Ignore)]
    public int? Offset {get; set;}
    
    [JsonProperty(NullValueHandling = NullValueHandling.Ignore)]
    public int? Limit {get; set;}
    
    public int Size {get; set;}
    
    [JsonProperty(NullValueHandling = NullValueHandling.Ignore)]
    public Link First {get; set;}
    
    [JsonProperty(NullValueHandling = NullValueHandling.Ignore)]
    public Link Previous {get; set;}
    
    [JsonProperty(NullValueHandling = NullValueHandling.Ignore)]
    public Link Next {get; set;}
    
    [JsonProperty(NullValueHandling = NullValueHandling.Ignore)]
    public Link Last {get; set;}
    
    private static Link GetNextLink(Link self, int size, PagingOptions pagingOptions)
    {
      if (pagingOptions?.Limit == null) return null;
      if (pagingOptions?.Offset == null) return null;
      
      var limit = pagingOptions.Limit.Value;
      var offset = pagingOptions.Offset.Value;
      var nextPage = offset + limit;
      
      if (nextPage >= size)
      {
        return null;
      }
      
      var parameters = new RouteValueDictionary(self.RouteValues)
      {
        ["limi"] = limit,
        ["offset"] = nextPage,
      };
      
      var newLink = Link.ToCollection(self.RouteName, parameters);
      return newLink;
    }
  }
}
```
We create a second model as follows :

```
namespace App.Models
{
  public class PagingOptions
  {
    [Range(1, 99999, ErrorMessage="Offset must be greater than zero")]
    public int? Offset {get; set;}
    
    [Range(1, 100, ErrorMessage="Limit must be greater than 0 and less than 100")]
    public int? Limit {get; set;}
    
    public PagingOptions Replace(PagingOptions newer) 
    {
      return new PagingOptions {
        Offset = newer.Offset ?? this.Offset,
        Limit = newer.Limit ?? this.Limit,
      };
    }
  }
} 
```

Not that a part of the code is not included here. The above can be used in another class as follows

```
var pagedOptions = allOpenings.Skip(pagingOptions.Offset.Value).Take(pagingOptions.Limit.Value);
```

We can also add some default pagination as follows. In the appsettings.json file you can write :

```
  "DefaultPagingOptions":{
    "limit" : 25,
    "offset" : 0,
  }
```
Then you can add this to the ConfigureServices method in the Startup.cs file as follows  :

```
services.Comfigure<PagingOptions>(Configuration.GetSection("DefaultPagingOptions"));
```

We can configure the behavior of the Api when errors occur as follows:

```
services.Configure<ApiBehaviorOptions>(options =>
{
  options.InvalidModelStateResponseFactory = context =>
  {
    var errorResponse = new ApiError(context.ModelState);
    return new BadRequestObjectResult(errorResponse);
  };
};
```

Now we want to add a navigation to our paged collection responses. Next we want to add sorting behavior so we will need to add the following code :

```
namespace App.Models
{
  public class Room : Resource
  {
    [Sortable]
    [Searchable]
    public string Name {get;set;}
    
    [Sortable(Default = true)]
    [Searchable]
    public decimal Rate {get; set;}
  }
}
```
Next we write the following code. IValidatableObject is an interface that holds methods that will be called when it's trying to validate the parameters passed into the controllers. The Apply method after is used to apply the sort options to a database query.

````
namespace App.Models
{
  public class SortOptions<T, TEntity> : IValidateObject
  {
    public string[] OrderBy {get; set;}
    
    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
      var processor = new SortOptionsProcessor<T, TEntity>(OrderBy);
      
      var validTerms = processor.GetValidTerms().Select(x => x.Name);
      var invalidTerms = processos.GetAllTerms().Select(x => x.Name )
      .Except(validTerms, StringComparer.OrdinalIgnoreCase);
      
      foreach (var term in invalidTerms)
      {
        yield return new ValidationResult($"Invalid sort term '{term}'.", new[] {nameof(OrderBy)});
      }
    }
    
    public IQueryable<TEntity> Apply(IQueryable<TEntity> query)
    {
    
    }
  }
}
````

We can use the above as follows 

```
public async Task<PagedResults<Room>> GetRoomsAsync(
  PagingOptions pagingOptions,
  SortOptions<Room, RoomEntity> sortOptions) { 
  
  IQueryable<RoomEntity> query = _context.Rooms;
  query = sortOptions.Apply(query);
  ... }
)
```

The function used is written below : 

```
namespace App.Infrastructure
{
  public class SortOptionsProcessor<T, TEntity>
  {
    private readonly string[] _orderBy;
    
    public SortOptionsProcessor(string[] orderBy)
    {
      _orderBy = orderBy;
    }
    
    public IEnumerable<SortTerm> GetAllTerms()
    {
      if (_orderBy == null) yield break;
      
      foreach (var term in _orderBy)
      {
        if (string.IsNullOrEmpty(term)) continue;
        
        var tokens = term.Split(' ');
        
        it(tokens.Length == 0)
        {
          yield return new SortTerm {Name = term};
          continue;
        }
        
        var descengin = tokens.Length > 1 && tokens[1].Equals("desc", StringComparison.OrdinalIgnoreCase)
        
        yield return new SortTerm
        {
          Name = tokens[0],
          Descending = descending,
        };
      }
    }
    
    public IEnumerable<SortTerm> GetValidTerms()
    {
      var queryTerms = GetAllTerms.ToArray();
      if (!queryTerms.Any()) yield break;
      var declaredTerms = GetTermsFromModel();
      
      foreach(var term in queryTerms)
      {
        var declaredTerm = declaredTerms.SingleOrDefault(x => x.Name.Equals(term.Name, StringComparison.OrdinalIgnoreCase));
        if (declaredTerm == null) continue;
        
        yield return new SortTerm
        {
          Name = declaredTerm.Name,
          Descending = term.Descending,
        }
      }
    }
    
    private static IEnumerable<sortTerm> GetTermsFromModel()
    => typeof(T).GetTypeInfo().DeclaredProperties.Where(p => p.GetCustomAttribute<SortableAttribute>().Any())
    .Select(p => new SortTerm{Name = p.Name});
    
    public IEnumerable<SortTerm> GetValidTerms()
    {
      var queryTerms = GetAllTerms().ToArray();
      if (!queryTerms.Any())
      {
        yield break;
      }
      
      var declaredTerms = GetTermsFromModel();
      
      foreach(var term in queryTerms)
      {
        var declaredTerm = declaredTerms
          .SingleOrDefault(x => x.Name.Equals(term.Name, StringComparison.OrdinalIgnoreCase));
        
        if (declaredTerm == null) continue;
        
        yield return new SortTerm
        {
          Name = declaredTerm.Name,
          Descending = term.Descengin,
        }
      }
    }
    
    private static IEnumerable<SortTerm> GetTermsFromModel() => typeof(T).GetTypeInfo().DeclaredProperties
    .Where(p => p.GetCustomAttributes<SortableAttribute>().Any()).Select(p => new SortTerm{ Name = p.Name})
    
  }
  
  public class SortTerm
  {
    public string Name {get; set;}
    
    public bool Descending {get; set; }
  }
}
```

Then some more code is added in order to create a search interface and create a form to retrieve and delete resources. Now to save data in the cache you may write : `[ResponseCache(Duration = 84600)]`. You can return the ETag header in the cache by using for example the following code :

```
var serialized = JsonConvert.SerializeObject(this);
return Md5Hash.ForString(serialized);
```

Authentication checks the user is says who he says he is. Basic authentication checks the username and the password. Bearer authentication issues a token from the server. The Digest authentication makes a hash from the request data. In the OpenID syste, the client system send the username and passwrod, receives a token and sends an authorization to the server. We can create an authentication scheme as follows :

```
namespace App.Models
{
  public class UserEntity : IdentityUser<Guid>
  {
    public string Firstname {get; set;}
    
    public string Lastname {get; set;}
    
    public DateTimeOffset CreatedAt {get; set;}
  }
  
  public class UserRoleEntity : IdentityRole<Guid>
  {
    public UserRoleEntity() : base()
    {
    
    }
    
    public UserRoleEntity(string roleName) : base(roleName)
    {
    
    }
  }
}
```

Now in the startup file you want to add these so we write :

```
private static void AddIdentityCoreServices(IServiceCollection services)
{
  var builder = services.AddIdentityCore<UserEntity>();
  builder = new IdentityBuilder(
    builder.UserType,
    typeof(UserRoleEntity),
    builder.Services);
  
  builder.AddRoles<UserRoleEntity>()
         .AddEntityFrameworkStores<AppDbContext>()
         .AddDefaultTokenProviders()
         .AddSignInManager<SignInManager<UserEntity>>();
}
```

Then you can also add a test user in the SeedData file :

```
private static async Task addTestUsers(
  RoleManager<UserRoleEntity> roleManager,
  UserManager<UserEntity> userManager)
{
  var dataExists = roleManager.Roles.Any() || userManager.Users.Any();
  if (dataExists) { return; }
  
  await roleManager.CreateAsync(new UserRoleEntity("Admin"));
  
  var user = new UserEntity{
    Email = "admin@app.local"
    Username = "admin@app.local"
    Firstname = "admin",
    Lastname = "tester",
    CreatedAt = DateTimeOffset.UtcNow
  }
  
  await userManager.CreateAsync(user, "supersecrete123||");
  await userManager.AddToRolesAsync(user, "Admin");
  await userManager.UpdateAsync(user);
}
```

Now we need to implement this method as follows :

```
public static async Task InitializeAsync(IServiceProvider services)
{
  await AddTestUsers(
    services.GetRequiredService<RoleManager<UserRoleEntity>>(),
    services.GetRequiredService<UserManager<UserEntity>>());
    
    ...
}
```

Along with the Users file, there is also a UsersController and DefaultUsersController. One example of a token authetication manager is OpenIddict, this is a lightweight OpenID Connect authorization server that plugs into ASP .NET Core identity and identity framework core. We need to install OpenIddict and OpenIddict.EntityFrameworkCore from the nuget package manager. We need to configure this in the services as follows :

```
services.Configure<IdentityOptions>(options =>
{
  options.ClaimsIdentity.UsernameClaimType = OpenIdConnectConstants.Claims.Name;
  options.ClaimsIdentity.UserIdclaimType = OpenIdConnectConstants.Claims.Subject;
  options.ClaimsIdentity.RoleClaimType = OpenIdConnectConstance.Claims.Role;
});

services.AddAuthentication(options => {
  options.DefaultScheme = OpenIddictValidationDefaults.AuthenticationScheme;
})

...

app.UseAuthentication();
```

Policies are groups of authorization requirements, like role checks or claim checks.
