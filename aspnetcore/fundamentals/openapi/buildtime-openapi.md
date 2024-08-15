---
title: Generate OpenAPI documents at build time
author: captainsafia
description: Learn how to generate OpenAPI documents in your application's build step
ms.author: safia
monikerRange: '>= aspnetcore-6.0'
ms.custom: mvc
ms.date: 8/13/2024
uid: fundamentals/openapi/buildtime-openapi
---
<!-- backup writer.sms.author: tdykstra and rick-anderson -->

# Generate OpenAPI documents at build-time

In a typical web apps, OpenAPI documents are generated at run-time and served via an HTTP request to the app server.

Generating OpenAPI documentation during the app's build step can be useful for documentation that is:

- Committed into source control
- Used for spec-based integration testing
- Served statically from the web server

To add support for generating OpenAPI documents at build time, install the [`Microsoft.Extensions.ApiDescription.Server`](https://www.nuget.org/packages/Microsoft.Extensions.ApiDescription.Server) NuGet package:

### [Visual Studio](#tab/visual-studio)

Run the following command from the **Package Manager Console**:

 ```powershell
 Install-Package Microsoft.Extensions.ApiDescription.Server -IncludePrerelease
```

### [.NET CLI](#tab/net-cli)

Run the following command in the directory that contains the project file:

```dotnetcli
dotnet add package Microsoft.Extensions.ApiDescription.Server --prerelease
```

---

The `Microsoft.Extensions.ApiDescription.Server` package automatically generates the Open API document(s) associated with the app during build and populates them into the app's output directory:

Consider a template created API app named `MyTestApi`:

### [Visual Studio](#tab/visual-studio)

The output tab of Visual Studio includes the output similar to the following:

```text
1>Generating document named 'v1'.
1>Writing document named 'v1' to 'MyProjectPath\obj\MyTestApi.json'.
```

### [.NET CLI](#tab/net-cli)

The following commands build the app and display the generated OpenAPI document:

```cli
$ dotnet build
$ cat obj/MyTestApi.json
```

---

The generated `obj/{MyProjectName}.json` file contains the [OpenAPI version, title,  endpoints, and more](https://learn.openapis.org/specification/structure.html). The first few lines of obj/MyTestApi.json file:

:::code language="json" source="~/fundamentals/openapi/samples/9.x/BuildTime/csproj/MyTestApi.json" id="snippet_1" highlight="4-5":::

## Customize build-time document generation

### Modify the output directory of the generated Open API file

By default, the generated OpenAPI document is generated in the app's output directory. To modify the location of the generated file, set the target path in the `OpenApiDocumentsDirectory` property in the project file:

<!-- Original had
   <OpenApiDocumentsDirectory>./</OpenApiDocumentsDirectory>
Which generates misleading error: Missing required option '--project'.
-->

:::code language="xml" source="~/fundamentals/openapi/samples/9.x/BuildTime/csproj/MyTestApi.csproj.html" id="snippet_1" highlight="2":::

The value of `OpenApiDocumentsDirectory` is resolved relative to the project file. Using the `.` value in the preceding markup generates the OpenAPI document in the same directory as the project file. To generate the OpenAPI document in a different directory, provide the path relative to the project file. In the following example, the OpenAPI document is generated in the `MyOpenApiDocs` directory, which is a sibling directory of the project directory:

:::code language="xml" source="~/fundamentals/openapi/samples/9.x/BuildTime/csproj/MyTestApi.csproj.html" id="snippet2" highlight="2":::

### Modify the output file name

By default, the generated OpenAPI document has the same name as the app's project file. To modify the name of the generated file, set the `--file-name` argument in the `OpenApiGenerateDocumentsOptions` property:

:::code language="xml" source="~/fundamentals/openapi/samples/9.x/BuildTime/csproj/MyTestApi.csproj.html" id="snippet_file_name" highlight="2":::

In development mode, the preceding markup generates the `obj/my-open-api.json` file.

### Select the OpenAPI document to generate

Some apps may be configured to generate multiple OpenAPI documents, for example:

* For different versions of an API.
* To distinguish between public and internal APIs.

By default, the build-time document generator creates files for all documents that are configured in an app. To generate for a single document only, set the `--document-name` argument in the `OpenApiGenerateDocumentsOptions` property:

:::code language="xml" source="~/fundamentals/openapi/samples/9.x/BuildTime/csproj/MyTestApi.csproj.html" id="snippet_doc_name":::

<!--
What's the equivalent  of 
 app.UseSwaggerUI(c =>
    {
        c.SwaggerEndpoint("/swagger/v1/swagger.json", "Public API v1");
        c.SwaggerEndpoint("/swagger/v2/swagger.json", "Internal API v2");
    });
-->

## Customize run-time behavior during build-time document generation

The OpenAPI document generation at build-time works by starting the app’s entrypoint with a temporary background server. This is a requirement to produce accurate OpenAPI documents because all information in the OpenAPI document can't be statically analyzed. Because the app's entrypoint is invoked, any logic in the apps' startup is invoked. The app's entrypoint includes code that injects services into the DI container or reads from configuration. In some scenarios, it's necessary to restrict the code paths that run when the app's entry point is being invoked from build-time document generation. These scenarios include:

- Not reading from certain configuration strings.
- Not registering database-related services.

In order to restrict these code paths from being invoked by the build-time generation pipeline, they can be conditioned behind a check of the entry assembly:

```csharp
using System.Reflection;

var builder = WebApplication.CreateBuilder();

if (Assembly.GetEntryAssembly()?.GetName().Name != "GetDocument.Insider")
{
  builder.Services.AddDefaults();
}
```