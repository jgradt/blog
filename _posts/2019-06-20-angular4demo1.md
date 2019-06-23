---
layout: post
title: Angular client with Asp.Net Core WebApi backend (Part 1)
categories: repos angular4demo
permalink: /:categories/part1.html
---

## Introduction

I created this solution to help me learn more about Angular and how I could use it to create a modern website.  I decided to write both a front-end SPA that runs in the browser as well as a backend api that would serve up the data through REST endpoints.  I had already done a bit of work with .Net, so I created the backend with Asp.Net Core WebApi.  This post covers the code and design of the **WebApi** project.

*NOTE: I actually wrote the code for this repository a while ago, but I thought I'd write this post to document a bit of how it works*

## Overview

* Front-end SPA is created with Angular 4 written in Typescript
* Back-end REST service endpoints are created with Asp.Net Core WebApi
* The data exists in an Entity Framework Core In-memory database.  Random, fake data is generated at startup.
* Security for the service endpoints use Json Web Tokens
* A unit test project to test the WebApi code also exists as part of the solution.
* The database schema has two tables with Customers and Orders.

## The Code

### The Data

To begin with, lets define and set up the database schema and the data.  The purpose of this solution is to manage the data for customers and orders.  Therefore, we will need to start by creating a `DbContext` with those two tables in the schema along with their corresponding Entity classes.

```csharp

public class DemoDbContext : DbContext
{
    public DemoDbContext(DbContextOptions<DemoDbContext> options)
        : base(options)
    {
    }

    public DbSet<Customer> Customers { get; set; }
    public DbSet<Order> Orders { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Order>()
            .HasOne(o => o.Customer)
            .WithMany(c => c.Orders)
            .OnDelete(DeleteBehavior.Cascade); ;
    }
}

```

```csharp

public class Customer : IDataEntity
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
    public DateTime CreatedDate { get; set; }
    public DateTime LastUpdatedDate { get; set; }

    public List<Order> Orders { get; set; }
}

```

The classes in the "Entity" folder are the Entity Framework classes corresponding to the way the data is represented in the database.  

We also need some data transfer objects.  These are located in the "Dto" folder of the project and represent the data structure that will be passed to and from the client.  Not all of the fields in the Entity classes need to match exactly to the fields in the Dto classes.

```csharp

public class CustomerDto
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
}

// Fluent Validation
public class CustomerDtoValidator : AbstractValidator<CustomerDto>
{
    public CustomerDtoValidator()
    {
        RuleFor(x => x.Id).NotNull();
        RuleFor(x => x.FirstName).NotNull().Length(1, 50);
        RuleFor(x => x.LastName).NotNull().Length(1, 50);
        RuleFor(x => x.Email).EmailAddress().MaximumLength(150);
    }
}

// AutoMapper
public class CustomerAutoMapProfile : Profile
{
    public CustomerAutoMapProfile()
    {
        CreateMap<Customer, CustomerDto>()
            .ForSourceMember(m => m.CreatedDate, opt => opt.Ignore())
            .ForSourceMember(m => m.LastUpdatedDate, opt => opt.Ignore())
            .ForSourceMember(m => m.Orders, opt => opt.Ignore())
            .ReverseMap();
    }

}

```

I am using AutoMapper to map data between the Entity and Dto classes.  I'm using Fluent Validation to validate the incoming data on the Dtos.  I needed additional classes to enable fluent validation and mapping functionality, so I put all of these in the same file as the Dtos so that they would be easy to maintain.

### Repository Classes

Next, I want to create some repository classes to give standard methods for communicating with the backend data store.  This will allow me to perform all CRUD operations and even provide data retrieval with paging.

The repository classes are very simple in their implementations.  That is because they inherit from a base class that provides the same basic operations to each repository.

```csharp

public interface IBaseDataRepository<TEntity>
    where TEntity: class, IDataEntity
{
    List<TEntity> GetAll(Expression<Func<TEntity, bool>> filter = null, Func<IQueryable<TEntity>,
        IOrderedQueryable<TEntity>> orderBy = null, string[] includeProperties = null);
    Task<List<TEntity>> GetAllAsync(Expression<Func<TEntity, bool>> filter = null, Func<IQueryable<TEntity>, 
        IOrderedQueryable<TEntity>> orderBy = null, string[] includeProperties = null);
    PagedData<TEntity> GetPaged(int pageIndex, int pageSize, Expression<Func<TEntity, bool>> filter = null,
        Func<IQueryable<TEntity>, IOrderedQueryable<TEntity>> orderBy = null, string[] includeProperties = null);
    Task<PagedData<TEntity>> GetPagedAsync(int pageIndex, int pageSize, Expression<Func<TEntity, bool>> filter = null, 
        Func<IQueryable<TEntity>, IOrderedQueryable<TEntity>> orderBy = null, string[] includeProperties = null);
    TEntity GetById(int id);
    Task<TEntity> GetByIdAsync(int id);
    int GetCount(Expression<Func<TEntity, bool>> filter = null);
    Task<int> GetCountAsync(Expression<Func<TEntity, bool>> filter = null);
    bool GetExists(Expression<Func<TEntity, bool>> filter = null);
    Task<bool> GetExistsAsync(Expression<Func<TEntity, bool>> filter = null);
    Task<TEntity> AddAsync(TEntity entity);
    Task UpdateAsync(int id, TEntity entity);
    Task DeleteAsync(int id);
    int Save();
    Task<int> SaveAsync();
}

public class BaseDataRepository<TEntity> : IBaseDataRepository<TEntity>
    where TEntity: class, IDataEntity
{
    // ... implementation of methods is here
}

```

To create a new repository, simply inherit from the `BaseDataRepository` class.  I have also created an interface for each repository that inherits from `IBaseDataRepository`.  These interfaces will be used along with a dependency injection container to inject the appropriate repository classes into controllers as needed.

```csharp

public interface ICustomerRepository : IBaseDataRepository<Customer>
{
}

public class CustomerRepository : BaseDataRepository<Customer>, ICustomerRepository
{

    public CustomerRepository(DemoDbContext demoDbContext) : base(demoDbContext)
    {
    }

    public override void SetDataForUpdate(Customer sourceEntity, Customer destinationEntity)
    {
        destinationEntity.FirstName = sourceEntity.FirstName;
        destinationEntity.LastName = sourceEntity.LastName;
        destinationEntity.Email = sourceEntity.Email;
    }

}

```

In addition, the base repository provides a way to return paged data in case there are too many records to be returned at once.  I have created a separate Dto for this.  It contains the information relating to the page and total number of items as well as the subset of items themselves.

```csharp

public class PagedData<TData>
{
    public int PageIndex { get; set; }
    public int PageSize { get; set; }
    public int TotalItems { get; set; }
    public List<TData> Items { get; set; }
}

```

### Api Controllers

Now we need some Api Controllers that can accept Http requests, call methods on the repository classes, and return the data to the client.

```csharp

[Authorize(Roles = "Admin")]
[Route("api/customers")]
public class CustomersController : Controller
{
    private readonly ICustomerRepository customerRepository;
    private readonly IMapper mapper;

    public CustomersController(ICustomerRepository customerRepository, IMapper mapper)
    {
        this.customerRepository = customerRepository;
        this.mapper = mapper;
    }

    [HttpGet]
    public async Task<PagedData<CustomerDto>> Get(int pageIndex = 0, int pageSize = 10)
    {
        var data = await customerRepository.GetPagedAsync(pageIndex, pageSize);
        var mappedData = new PagedData<CustomerDto>()
        {
            PageIndex = data.PageIndex,
            PageSize = data.PageSize,
            TotalItems = data.TotalItems,
            Items = mapper.Map<List<CustomerDto>>(data.Items)
        };

        return mappedData;
    }

    [HttpGet("{id}", Name = "GetCustomer")]
    public async Task<IActionResult> Get(int id)
    {
        var customer = await customerRepository.GetByIdAsync(id);
        if(customer == null)
        {
            return NotFound();
        }

        var mappedData = mapper.Map<CustomerDto>(customer);

        return new ObjectResult(mappedData);
    }

    // ... more methods here for other Http requests
}

```

### Security

Json Web Tokens (JWTs) are used for authorization.  A REST endpoint exists on the `TokenController` that accepts login information and will return a JWT.  The token contains information such as the issuer and expiration date of that token.  It also contains a list of claims relevant to that user such as their name and authorized roles for calling the api endpoints.  The claims information returned in the JWT can be customized as desired.  In a real application, the login information would be verified with the user name and password against a backend data store.  Once the client has obtained a token from the api, they will need to include that token in the Authorization header of each web request to the REST endpoints.

```csharp

private object GenerateToken(string username)
{
    var secretKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(appConfig.Settings.Jwt.SecretKey));

    //TODO: send back retrieved claims
    var claims = new Claim[]
    {
        new Claim(ClaimTypes.Name, "John"),
        new Claim(ClaimTypes.Role, "Admin"),
        new Claim(JwtRegisteredClaimNames.Email, "john.doe@blah.com")
    };

    var token = new JwtSecurityToken(
        issuer: appConfig.Settings.Jwt.Issuer,
        audience: appConfig.Settings.Jwt.Audience,
        claims: claims,
        notBefore: DateTime.Now,
        expires: DateTime.Now.AddMinutes(20),
        signingCredentials: new SigningCredentials(secretKey, SecurityAlgorithms.HmacSha256)
    );

    string jwtToken = new JwtSecurityTokenHandler().WriteToken(token);

    return jwtToken;
}

```
Each of the controllers that requires security has an `[Authorize]` attribute.  And the `StartupExtensions` class has some additional code to configure the JWT at startup.

```csharp

public static void AddJwt(this IServiceCollection services, AppSettings appSettings)
{
    services.AddAuthentication(options =>
    {
        options.DefaultAuthenticateScheme = "JwtBearer";
        options.DefaultChallengeScheme = "JwtBearer";
    })
    .AddJwtBearer("JwtBearer", jwtBearerOptions =>
    {
        jwtBearerOptions.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(appSettings.Jwt.SecretKey)),

            ValidateIssuer = true,
            ValidIssuer = appSettings.Jwt.Issuer,

            ValidateAudience = true,
            ValidAudience = appSettings.Jwt.Audience,

            ValidateLifetime = true, //validate the expiration and not before values in the token

            ClockSkew = TimeSpan.FromMinutes(5) //5 minute tolerance for the expiration date
        };
    });
}

```

If an attempt is made to access the endpoint without a valid token in the request header, then a 401 Unauthorized response will be returned.

### Startup Configuration

There are a number of different statements that needed to be written in the `Configure` and `ConfigureServices` methods of the `Startup` class.  These statements register classes for dependency injection, configure the logging provider, seed the database on startup, and add/configure other services such as Json Web Tokens, Fluent Validation, AutoMapper, etc.  

```csharp

// This method gets called by the runtime. Use this method to add services to the container.
public void ConfigureServices(IServiceCollection services)
{
    var appSettings = Configuration.GetSection("App").Get<AppSettings>();
    services.Configure<AppSettings>(Configuration.GetSection("App"));

    services.AddJwt(appSettings);

    services.AddCors();

    services.AddDbContext<DemoDbContext>(opt => opt.UseInMemoryDatabase("demoDb"));

    services.AddMvc(opt =>
    {
        opt.Filters.Add(typeof(ValidatorActionFilter));
    })
    .AddFluentValidation(cfg => { cfg.RegisterValidatorsFromAssemblyContaining<Startup>(); }); ;

    services.AddAutoMapper();

    services.AddSwaggerGen(c =>
    {
        c.SwaggerDoc("v1", new Info { Title = "WebApiDemo API", Version = "v1" });

        c.AddSecurityDefinition("Bearer", new ApiKeyScheme
        {
            Description = "JWT Authorization header using the Bearer scheme. Example: \"Authorization: Bearer {token}\"",
            Name = "Authorization",
            In = "header",
            Type = "apiKey"
        });
    });

    //register types for DI
    services.AddScoped<ICustomerRepository, CustomerRepository>();
    services.AddScoped<IOrderRepository, OrderRepository>();
    services.AddSingleton<AppConfig, AppConfig>();

}

```

### Testing the endpoints with Swagger UI

Swagger has made it very easy to test the endpoints.  Adding it to the project is really easy with a few nuget packages and a little bit of startup code (see above).

Since the endpoints are protected with the security of JWTs, you'll need to get a token and authorize first.  You just need to post to the `api/token` endpoint of the `TokenController` to begin with.  Take the resulting token and use it when you click the Authorize button at the top of the Swagger UI screen.  Now you can test the endpoints and see the results.

![screenshot]({{ '/assets/angular4demo/images/swagger-ui-get-customer.png' | relative_url }})

## Conclusion

Using Asp.Net Core and WebApi provide a nice way to make some REST endpoints that manage data in a data store.  Once the basic architecture is written, it is relatively straightforward and easy to add additional endpoints to manage/query the existing data or any new tables you might want to include in your schema.

## See also
* <https://docs.microsoft.com/en-us/ef/core/>
* <https://automapper.readthedocs.io/en/latest/Getting-started.html>
* <https://github.com/JeremySkinner/FluentValidation>
* <https://cpratt.co/truly-generic-repository/>
* <https://jwt.io/>
