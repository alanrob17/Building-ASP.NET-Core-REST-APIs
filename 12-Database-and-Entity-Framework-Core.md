# Database and Entity Framework Core

## Install Entity Framework Core

In Package Manager console install:

> dotnet add package Microsoft.EntityFrameworkCore      
>       
> dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL      
>       
> dotnet add package Microsoft.EntityFrameworkCore.Design       
>       
> dotnet tool install --global dotnet-ef        
>       
> dotnet tool update --global dotnet-ef

### Database provider

> dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL

This is the **PostgreSQL** provider for Entity Framework Core. Entity Framework Core on its own does not know how to talk to any specific database. It needs a provider to do that. Think of it as a translator between Entity Framework Core and PostgreSQL.

This package tells Entity Framework Core how to connect to PostgreSQL, how to generate PostgreSQL-compatible SQL, and how to handle PostgreSQL-specific data types.

If we were using a different database, we would simply install a different provider.

### Entity Framework design tools

> dotnet add package Microsoft.EntityFrameworkCore.Design

This package contains the design tools for Entity Framework Core. Most importantly, the tools we need to create and manage migrations. It is only needed during development, not in production.

### Entity Framework Core command line tool

> dotnet tool install --global dotnet-ef

This installs the Entity Framework Core command-line tool, which we will use to manage migrations. 
For example, to create a new migration, or apply it to the database. 

This is a global tool, meaning it is installed once on our machine and available in any project.

The Entity Framework Core design package we installed earlier works together with this tool. The package provides the code needed to generate migrations, while .NET Entity Framework is the tool that actually runs the commands.

## DbContext

A ``DbContext`` is the central class in Entity Framework Core. It is our main point of contact with the database. It represents an active session with the database, and through it we can query data, save new records, update existing ones and delete them.

Every operation in Entity Framework Core goes through a ``DbContext``.

Let's create one.

Add a new folder in our project and name it **DataAccess**.

Inside, I will create a new class called ``LibraryDbContext``. We typically create one ``DbContext`` per database. Since in our project we will only use one database, this will be our only ``DbContext``.

To make this class a ``DbContext``, we need it to inherit from the ``DbContext`` base class.

We must also provide a constructor that takes one parameter, ``DbContextOptions``.

```bash
    public class LibraryDbContext : DbContext
    {
        public LibraryDbContext(DbContextOptions<LibraryDbContext> options) : base(options)
        {
        
        }
    ...
```

``DbContextOptions`` carries all the configuration our ``DbContext`` needs.

Things like which database provider to use and the connection string. A connection string is simply a piece of text that tells Entity Framework Core where the database is and how to connect to it.

We pass the options to the base class constructor so that Entity Framework Core can read them.

Notice that ``DbContextOptions`` is a generic class. And we pass our own class, ``LibraryDbContext``, as the type parameter.

This is how Entity Framework Core knows which ``DbContext`` this configuration belongs to, which could become important when an application has more than one ``DbContext``.

### DbSet

Now, for each entity that will be mapped to a table in the database, we define a ``DbSet``.

A ``DbSet`` represents a table in the database.

```bash
    public DbSet<Category> Categories => Set<Category>();
    public DbSet<Author> Authors => Set<Author>();
```

It is what we query and operate on. When we want all ``Authors``, we go through this property.

The **Set of T** method on the right side is what actually creates and returns that set.

This returns all Authors. Don't be concerned about returning all this data it is a very lightweight operation.

Entity Framework Core caches these Sets internally, so nothing heavy is happening each time.

You may also come across a different syntax for this in other projects.

```bash
    public DbSet<Category> Categories { set; get;};
    public DbSet<Author> Authors { set; get;};
```

The difference is simple. Our approach with **Set of T** makes the property **read-only**. There is no public setter, so nothing outside of this class can overwrite it. This version exposes a public setter, which is generally unnecessary. Functionally, both approaches work the same way.

### Program.cs

We also need to make some changes in the ``Program.cs`` class.

First, we register the ``DbContext``.

```bash
    builder.Services.AddControllers();
    builder.Services.AddDbContext<LibraryDbContext>(options =>
    {
        options.UseNpgsql(builder.Configuration.GetConnectionString("LibraryDb"));
    });
    ...
```

``AddDbContext`` registers our ``DbContext`` with the application, so that it is available throughout the entire project.

``Options.UseNpgsql`` tells Entity Framework Core to use PostgreSQL as the database, using the provider we installed earlier.

This part retrieves the connection string from the configuration. That is from ``AppSettings.json``.

The key we are looking for is ``LibraryDB``.

Complete ``LibraryDbContext.cs``:

```bash
namespace LibraryManagement.Api.DataAccess
{
    public class LibraryDbContext : DbContext
    {
        public LibraryDbContext(DbContextOptions<LibraryDbContext> options) : base(options)
        {
        
        }

        public DbSet<Category> Categories => Set<Category>();
        public DbSet<Author> Authors => Set<Author>();
    }
}
```

## appsettings.json

Every ASP.NET Core application comes with a configuration file called ``appsettings.json``.

This is where we store settings that our application needs to run. Things like login configuration, feature flags or connection strings. It is a simple JSON file that lives in the root of our project.

If we open it, we can see it already has some content generated automatically when we created the project.

Let's take a quick look at what is in there.

The **logging** section controls how much information the application writes to the log output.

*Allowed hosts* set to asterisk means that the application will accept requests from any host.

Next to ``appsettings.json``, we also have a file called ``appsettings.development.json``.

Let's talk about why that exists. Applications typically run in different environments.

For example, on a developer's machine during development and on a server in production. These environments often need different settings.

![Environment settings](assets/images/environments.jpg "Environment settings")

A development database is not the same as a production database. Logging might be more detailed locally than on a live server. .NET handles these through environment-specific configuration files.

``Appsettings.json`` contains the base configuration that applies everywhere.

``Appsettings.development.json`` contains settings specific to the development environment. And they override the base file when the application runs locally. The base file is loaded first and the environment-specific file is loaded on top. If the same key appears in both, the **environment-specific** value wins.

So how does .NET know which environment is it running in? Well, it reads an environment variable called ASP.NET CoreEnvironment.

When we run the application from Visual Studio or the .NET CLI locally, this is automatically set to development. Which is why ``appsettings.development.json`` gets picked up.

![Environment types](assets/images/environment-types.jpg "Environment types")

On a production server, this would typically be set to production.

**Environment variables** are another layer on top of this configuration system. They are key-value pairs set at the operating system level, outside our application.

They sit at the top of the configuration hierarchy, meaning they override everything. Both ``appsettings.development.json`` and the environment-specific file. This is how production secrets are typically handled in professional setups.

Sensitive values, like connection strings or API keys, never live in a file. They are injected as environment variables by the hosting platform.

## Connection strings

A connection string is a piece of text that tells our application where the database is and how to connect to it. It contains things like the host address, the port, the database name and the credentials. 

Where should the connection string go?

We could put it in ``appsettings.json``, but that file is meant to contain **base configuration** that applies to all environments. Settings that are safe and universal.

A connection string is neither of those things. It is local, it is environment-specific and it contains credentials.

**Appsettings.development.json** is the better place. It clearly communicates that this setting belongs to the development environment only.

In production, the connection string would never come from a file. It would be injected as an environment variable by the hosting platform.

Keeping the connection string in the development file from the start builds the right habit.

Environment-specific secrets belong in environment-specific configuration.

### Adding a connection string section

```bash
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ConnectionStrings": {
    "LibraryDb": "Host=localhost;Port=15432;Database=librarydb;User Id=postgres;Password=london@1;"
  }
}
```

**Host** = localhost means the database is running on our own machine.

**Port** is the default PostgresQL port. The same one we mapped when we set up the Docker container.

The key **LibraryDb** here matches exactly what we used in the ``Program.cs`` file.

Let's make sure now this value can be read. 

Our Docker container:

> docker run --name library-postgres -e POSTGRES_PASSWORD=london@1 -e POSTGRES_DB=librarydb -p 15432:5432 -d postgres

Set a breakpoint on:

> if (app.Environment.IsDevelopment())

Run Debug:

Add a watch value:

> builder.Configuration.GetConnectionString("LibraryDb")

If successful you should be able to view the **connection string**.

With this in place, our application has everything it needs to connect to the database.

## Migrations
