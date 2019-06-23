---
layout: post
title: GraphQL.Net with Asp.Net Core and C# 
categories: repos graphqldemo
permalink: /:categories
---

## Introduction

I created this project in order to learn more about [GraphQL](https://graphql.org).  I decided to enhance an [existing project](https://github.com/jgradt/Angular4Demo/tree/master/WebApiDemo) I previously wrote that used Asp.net Core, Entity Framework, and many custom repository classes.  I used the [GraphQL.Net](https://github.com/graphql-dotnet/graphql-dotnet) nuget package and some online resources to help me do this.  To learn more about GraphQL and GraphQL.Net, see some of the additional resources at the bottom of this page.

## Overview

* This project contains REST and GraphQL Http endpoints built using Asp.net Core WebApi.  
* The REST endpoints provide CRUD functionality through requests with different Http verbs (GET, PUT, POST, DELETE). The objects returned from the endpoints are standardized data transfer objects.  
* The GraphQL endpoint provides query/retrieve functionality only (code for handling mutations has not been written).  The endpoint can return any validly shaped json object conforming to the schema.
* The underlying database schema is based on a customer order schema similar to Microsoft's Northwind sample database.  The data is randomly generated at startup.
* I based this on the code from another WebApi project I wrote [here](https://github.com/jgradt/Angular4Demo/tree/master/WebApiDemo).  I copied that code into a new project and made appropriate updates and enhancements to enable GraphQL queries.

## The Code

The code to implement the GraphQL functionality is primarily in a folder named "GraphQL" in the project

![screenshot]({{ site.baseurl }}/assets/graphqldemo/images/graphql-folder.png)

This implementation uses the GraphType First approach of GraphQL.Net in the above, highlighted classes.  Below is a sample of how I coded the `CustomerGraphType` and retrieved the data from the backend data source.  I already had coded some repository classes that could interact with my data source, so I decided to make use of those.  The repository classes can be injected into the `ObjectGraphType` classes and then used to retrieve the data without writing a lot of extra code.

```csharp

public class CustomerGraphType : ObjectGraphType<Customer>
{
    public CustomerGraphType(IOrderRepository orderRepository)
    {
        Name = "Customer";

        Field(x => x.Id, type: typeof(IdGraphType));
        Field(x => x.FirstName);
        Field(x => x.LastName);
        Field(x => x.Email);

        FieldAsync<ListGraphType<NonNullGraphType<OrderGraphType>>>(
            "orders",

            arguments: new QueryArguments(
                new QueryArgument<IntGraphType> { Name = "limit" }),

            resolve: async context =>
            {
                var numItems = context.GetArgument<int>("limit");
                numItems = numItems > 0 ? numItems : 10;

                var data = await orderRepository.GetPagedAsync(0, numItems, filter: o => o.CustomerId == context.Source.Id);

                return data.Items;
            }
        );
    }
}

```

Some additional code also exists in the Startup class of the project to configure dependency injection…

```csharp

// repository types
services.AddScoped<ICustomerRepository, CustomerRepository>();
services.AddScoped<IOrderRepository, OrderRepository>();
services.AddScoped<IOrderDetailRepository, OrderDetailRepository>();
services.AddScoped<IProductRepository, ProductRepository>();
services.AddScoped<ISupplierRepository, SupplierRepository>();

// GraphQL types
services.AddScoped<GraphQL.IDependencyResolver>(s => new GraphQL.FuncDependencyResolver(s.GetRequiredService));
services.AddScoped<GraphQL.Types.ISchema, GraphQLQuerySchema>();
services.AddGraphQL(x =>
{
    x.ExposeExceptions = true;
})
.AddGraphTypes(ServiceLifetime.Scoped);

```

… and to enable the iGraphQL UI…

```csharp

app.UseGraphiQl("/graphql");

```

## UI Data Exploration

*NOTE:  Data returned will be different each time the application is restarted because the data is randomly seeded in the database on startup.*

### REST / Swagger

A UI to explore and test the REST endpoints is included.  Once you run the project, it is located at `http://localhost:54618/swagger` and uses the Swagger UI to allow for testing the different endpoints.

![screenshot]({{ site.baseurl }}/assets/graphqldemo/images/swagger-ui.png)

### GraphiQL

A UI to explore and test the GraphQL endpoint is also included.  Once you run the project, it is located at `http://localhost:54618/graphql` and uses the GraphiQL UI to allow for testing and executing different GraphQL queries.  

#### Get an order by id with details

![screenshot]({{ site.baseurl }}/assets/graphqldemo/images/graphiql-order-by-id.png)

#### Get two customers using one query

![screenshot]({{ site.baseurl }}/assets/graphqldemo/images/graphiql-two-customers.png)

#### Get a Customer with Orders by customer id using an input variable

![screenshot]({{ site.baseurl }}/assets/graphqldemo/images/graphiql-customer-with-orders.png)

#### Get a product without supplying the id (expect an error)

![screenshot]({{ site.baseurl }}/assets/graphqldemo/images/graphiql-product-with-error.png)

#### Get first n customers

![screenshot]({{ site.baseurl }}/assets/graphqldemo/images/graphiql-get-customers.png)

## Authorization

Authorization on REST controllers is disabled, but it can be enabled by uncommenting the lines with the `[Authorize]` attributes.  When enabled, you must retrieve a token from the `TokenController` first and then authorize using that token through the Swagger UI.  Authorization code for GraphQL queries has not been written for this project.

## Conclusion

I tried to implement a fair amount of functionality here to help me learn all about the possibilities of working with GraphQL.  I actually really like it and think it provides a great additional endpoint for querying data.  I have not implemented all aspects however.  At the time of this writing, I have not implemented mutations to allow for updating data through the GraphQL endpoint.  I also have not yet implemented any real paging functionality through the GraphQL endpoint (although you can limit the number of results on many of the queries).  On the other hand, I had already written code previously for all of this through the REST endpoints, so this project still allows for full CRUD functionality and paging of data.  In this case, the GraphQL endpoint is just one extra way to query data in a very flexible manner without the need to provide REST endpoints for each specific scenario or to make multiple calls to the existing REST endpoints to get the data.

## See also

* <https://graphql.org/learn/>
* <https://github.com/graphql-dotnet/graphql-dotnet>
* <https://graphql-dotnet.github.io/docs/getting-started/introduction>
* <https://medium.com/volosoft/building-graphql-apis-with-asp-net-core-419b32a5305b>
* <https://developer.okta.com/blog/2019/04/16/graphql-api-with-aspnetcore>
