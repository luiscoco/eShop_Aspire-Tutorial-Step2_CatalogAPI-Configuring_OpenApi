# Building 'eShop' from Zero to Hero: Configuring CatalogAPI OpenAPI documentation

## 1. Load Nuget packages in "eShop.ServiceDefaults" project

### 1.1. Asp.Versioning.Mvc.ApiExplorer

**Purpose**: This package is part of the ASP.NET API Versioning project, specifically for applications using MVC (Model-View-Controller) architecture

**Functionality**: It provides tools to expose versioning information of your ASP.NET Core Web APIs through an IApiExplorer implementation, which allows you to generate API documentation for different API versions

**Use Case**: Useful when you want to manage and document multiple versions of your API endpoints and support versioned API documentation in tools like Swagger

### 1.2. Microsoft.AspNetCore.Open.Api

**Purpose**: This package provides support for **OpenAPI** (formerly known as **Swagger**) in ASP.NET Core applications

**Functionality**: It enables integration with the OpenAPI standard, which allows you to describe, produce, and consume RESTful APIs

This package includes various attributes and configuration options to define your API endpoints following OpenAPI specifications

**Use Case**: Essential when building APIs that need to be self-descriptive and documented in a standard format, making it easier for other developers to understand and consume your APIs

### 1.3. Scalar.AspNetCore

**Purpose**: This package is related to the Scalar project, which aims to manage large Git repositories, typically for enterprise solutions

**Functionality**: Scalar itself virtualizes file systems to make large Git repositories more manageable

Scalar.AspNetCore is likely an ASP.NET Core integration for managing interactions or services around Scalar repositories

**Use Case**: Useful for large enterprises dealing with significant Git repositories, aiming to streamline repository management and improve performance by integrating Scalar with ASP.NET Core

![image](https://github.com/user-attachments/assets/ff7cb645-65a4-47d7-b6d7-b927a7a23f35)

## 2. Create "ConfigurationExtensions.cs" file in "eShop.ServiceDefaults" project

![image](https://github.com/user-attachments/assets/5a1aae02-6e36-43aa-ad56-ae19e6f2f061)

This code defines a static method **GetRequiredValue** within the **ConfigurationExtensions** class, which is part of the **Microsoft.Extensions.Configuration namespace**

The purpose of this method is to retrieve a required configuration value from an **IConfiguration** object, which is commonly used in .NET applications for accessing application settings

```csharp
namespace Microsoft.Extensions.Configuration;

public static class ConfigurationExtensions
{
    public static string GetRequiredValue(this IConfiguration configuration, string name) =>
        configuration[name] ?? throw new InvalidOperationException($"Configuration missing value for: {(configuration is IConfigurationSection s ? s.Path + ":" + name : name)}");
}
```

## 3. Create "OpenApi.Extensions.cs" file in "eShop.ServiceDefaults" project

This setup provides a **customized OpenAPI** and **API versioning** setup for the application, enabling dynamic configurations, versioning, and conditional development settings for a more flexible API environment

This C# code snippet configures an **OpenAPI setup** for a .NET web application, specifically tailored for use with **API versioning** and an **OpenAPI/Swagger**-like setup

```csharp
using Asp.Versioning;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Routing;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Scalar.AspNetCore;

namespace eShop.ServiceDefaults;

public static partial class Extensions
{
    public static IApplicationBuilder UseDefaultOpenApi(this WebApplication app)
    {
        var configuration = app.Configuration;
        var openApiSection = configuration.GetSection("OpenApi");

        if (!openApiSection.Exists())
        {
            return app;
        }

        app.MapOpenApi();

        if (app.Environment.IsDevelopment())
        {
            app.MapScalarApiReference(options =>
            {
                // Disable default fonts to avoid download unnecessary fonts
                options.DefaultFonts = false;
            });
            app.MapGet("/", () => Results.Redirect("/scalar/v1")).ExcludeFromDescription();
        }

        return app;
    }

    public static IHostApplicationBuilder AddDefaultOpenApi(
        this IHostApplicationBuilder builder,
        IApiVersioningBuilder? apiVersioning = default)
    {
        var openApi = builder.Configuration.GetSection("OpenApi");
        var identitySection = builder.Configuration.GetSection("Identity");

        var scopes = identitySection.Exists()
            ? identitySection.GetRequiredSection("Scopes").GetChildren().ToDictionary(p => p.Key, p => p.Value)
            : new Dictionary<string, string?>();


        if (!openApi.Exists())
        {
            return builder;
        }

        if (apiVersioning is not null)
        {
            // the default format will just be ApiVersion.ToString(); for example, 1.0.
            // this will format the version as "'v'major[.minor][-status]"
            var versioned = apiVersioning.AddApiExplorer(options => options.GroupNameFormat = "'v'VVV");
            string[] versions = ["v1"];
            foreach (var description in versions)
            {
                builder.Services.AddOpenApi(description, options =>
                {
                    options.ApplyApiVersionInfo(openApi.GetRequiredValue("Document:Title"), openApi.GetRequiredValue("Document:Description"));
                    options.ApplyAuthorizationChecks([.. scopes.Keys]);
                    options.ApplySecuritySchemeDefinitions();
                    options.ApplyOperationDeprecatedStatus();
                    // Clear out the default servers so we can fallback to
                    // whatever ports have been allocated for the service by Aspire
                    options.AddDocumentTransformer((document, context, cancellationToken) =>
                    {
                        document.Servers = [];
                        return Task.CompletedTask;
                    });
                });
            }
        }

        return builder;
    }
}
```



## 4. Create "OpenApi.OptionsExtensions.cs" file in "eShop.ServiceDefaults" project


```csharp
using System.Text;
using Asp.Versioning.ApiExplorer;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc.ApiExplorer;
using Microsoft.AspNetCore.Mvc.ModelBinding;
using Microsoft.AspNetCore.OpenApi;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Primitives;
using Microsoft.OpenApi.Any;
using Microsoft.OpenApi.Models;

namespace eShop.ServiceDefaults;

internal static class OpenApiOptionsExtensions
{
    public static OpenApiOptions ApplyApiVersionInfo(this OpenApiOptions options, string title, string description)
    {
        options.AddDocumentTransformer((document, context, cancellationToken) =>
        {
            var versionedDescriptionProvider = context.ApplicationServices.GetService<IApiVersionDescriptionProvider>();
            var apiDescription = versionedDescriptionProvider?.ApiVersionDescriptions
                .SingleOrDefault(description => description.GroupName == context.DocumentName);
            if (apiDescription is null)
            {
                return Task.CompletedTask;
            }
            document.Info.Version = apiDescription.ApiVersion.ToString();
            document.Info.Title = title;
            document.Info.Description = BuildDescription(apiDescription, description);
            return Task.CompletedTask;
        });
        return options;
    }

    private static string BuildDescription(ApiVersionDescription api, string description)
    {
        var text = new StringBuilder(description);

        if (api.IsDeprecated)
        {
            if (text.Length > 0)
            {
                if (text[^1] != '.')
                {
                    text.Append('.');
                }

                text.Append(' ');
            }

            text.Append("This API version has been deprecated.");
        }

        if (api.SunsetPolicy is { } policy)
        {
            if (policy.Date is { } when)
            {
                if (text.Length > 0)
                {
                    text.Append(' ');
                }

                text.Append("The API will be sunset on ")
                    .Append(when.Date.ToShortDateString())
                    .Append('.');
            }

            if (policy.HasLinks)
            {
                text.AppendLine();

                var rendered = false;

                foreach (var link in policy.Links.Where(l => l.Type == "text/html"))
                {
                    if (!rendered)
                    {
                        text.Append("<h4>Links</h4><ul>");
                        rendered = true;
                    }

                    text.Append("<li><a href=\"");
                    text.Append(link.LinkTarget.OriginalString);
                    text.Append("\">");
                    text.Append(
                        StringSegment.IsNullOrEmpty(link.Title)
                        ? link.LinkTarget.OriginalString
                        : link.Title.ToString());
                    text.Append("</a></li>");
                }

                if (rendered)
                {
                    text.Append("</ul>");
                }
            }
        }

        return text.ToString();
    }

    public static OpenApiOptions ApplySecuritySchemeDefinitions(this OpenApiOptions options)
    {
        options.AddDocumentTransformer<SecuritySchemeDefinitionsTransformer>();
        return options;
    }

    public static OpenApiOptions ApplyAuthorizationChecks(this OpenApiOptions options, string[] scopes)
    {
        options.AddOperationTransformer((operation, context, cancellationToken) =>
        {
            var metadata = context.Description.ActionDescriptor.EndpointMetadata;

            if (!metadata.OfType<IAuthorizeData>().Any())
            {
                return Task.CompletedTask;
            }

            operation.Responses.TryAdd("401", new OpenApiResponse { Description = "Unauthorized" });
            operation.Responses.TryAdd("403", new OpenApiResponse { Description = "Forbidden" });

            var oAuthScheme = new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference { Type = ReferenceType.SecurityScheme, Id = "oauth2" }
            };

            operation.Security = new List<OpenApiSecurityRequirement>
            {
                new()
                {
                    [oAuthScheme] = scopes
                }
            };

            return Task.CompletedTask;
        });
        return options;
    }

    public static OpenApiOptions ApplyOperationDeprecatedStatus(this OpenApiOptions options)
    {
        options.AddOperationTransformer((operation, context, cancellationToken) =>
        {
            var apiDescription = context.Description;
            operation.Deprecated |= apiDescription.IsDeprecated();
            return Task.CompletedTask;
        });
        return options;
    }

    private static IOpenApiAny? CreateOpenApiAnyFromObject(object value)
    {
        return value switch
        {
            bool b => new OpenApiBoolean(b),
            int i => new OpenApiInteger(i),
            double d => new OpenApiDouble(d),
            string s => new OpenApiString(s),
            _ => null
        };
    }

    private class SecuritySchemeDefinitionsTransformer(IConfiguration configuration) : IOpenApiDocumentTransformer
    {
        public Task TransformAsync(OpenApiDocument document, OpenApiDocumentTransformerContext context, CancellationToken cancellationToken)
        {
            var identitySection = configuration.GetSection("Identity");
            if (!identitySection.Exists())
            {
                return Task.CompletedTask;
            }

            var identityUrlExternal = identitySection.GetRequiredValue("Url");
            var scopes = identitySection.GetRequiredSection("Scopes").GetChildren().ToDictionary(p => p.Key, p => p.Value);
            var securityScheme = new OpenApiSecurityScheme
            {
                Type = SecuritySchemeType.OAuth2,
                Flows = new OpenApiOAuthFlows()
                {
                    // TODO: Change this to use Authorization Code flow with PKCE
                    Implicit = new OpenApiOAuthFlow()
                    {
                        AuthorizationUrl = new Uri($"{identityUrlExternal}/connect/authorize"),
                        TokenUrl = new Uri($"{identityUrlExternal}/connect/token"),
                        Scopes = scopes,
                    }
                }
            };
            document.Components ??= new();
            document.Components.SecuritySchemes.Add("oauth2", securityScheme);
            return Task.CompletedTask;
        }
    }
}
```


## 5. Modify the "Pogram.cs" file in "Catalog.API" project 

This code snippet is setting up **API versioning** and **OpenAPI (Swagger)** integration in an ASP.NET Core application

```csharp
var withApiVersioning = builder.Services.AddApiVersioning();
builder.AddDefaultOpenApi(withApiVersioning);
```

By calling **AddApiVersioning**, you enable support for versioning your API endpoints

**AddDefaultOpenApi** is a method provided by some **OpenAPI/Swagger** integration packages, and it configures **OpenAPI** generation and documentation for the application

This code is configuring a versioned API named "Catalog" and setting up documentation for it using OpenAPI

This code helps in organizing **API versions** and making the **API self-documenting** for easier maintenance and usability

```csharp
app.NewVersionedApi("Catalog")
   .MapCatalogApiV1();
app.UseDefaultOpenApi();
```

**app.NewVersionedApi("Catalog")**: This line sets up a new API versioning scheme for a "Catalog" API

The **NewVersionedApi** method creates a new versioned API group called "Catalog", which allows you to manage multiple versions of this API (e.g., v1, v2) under the same logical name

This can help organize and structure different versions of endpoints within the "Catalog" API group

**.MapCatalogApiV1()**: This line maps specific routes or endpoints for the version 1 (v1) of the Catalog API

MapCatalogApiV1() is probably a custom method that sets up the routes/endpoints for version 1 of the Catalog API

For example, it could define routes like /api/v1/catalog for accessing resources within this version of the Catalog API

**app.UseDefaultOpenApi()**: This line adds support for OpenAPI (formerly known as Swagger) documentation in the application

By calling UseDefaultOpenApi, the application generates an OpenAPI specification (typically accessible via /swagger or /api/docs), providing a user-friendly way to view, test, and document the API's endpoints and operations

This makes it easier for developers and users to understand the API structure and how to interact with it

This is the whole **Program.cs** file

```csharp

var builder = WebApplication.CreateBuilder(args);

builder.AddServiceDefaults();

builder.AddApplicationServices();

builder.Services.AddProblemDetails();

var withApiVersioning = builder.Services.AddApiVersioning();

builder.AddDefaultOpenApi(withApiVersioning);

var app = builder.Build();

app.MapDefaultEndpoints();

app.NewVersionedApi("Catalog")
   .MapCatalogApiV1();

app.UseDefaultOpenApi();
app.Run();
```


## 6. 



