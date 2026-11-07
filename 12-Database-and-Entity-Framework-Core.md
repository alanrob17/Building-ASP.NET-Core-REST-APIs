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

Now that our application is connected to the database, we need to actually create the tables. 
This is where **migrations** come in.

A **migration** is the way Entity Framework Core synchronizes our classes with the database.

When we add a new entity, change a property, or add a relationship, the database knows nothing about those changes.

A migration is a file that describes what needs to change in the database to match our C# model and this is important to understand.

Until we generate and apply a migration, the database **will not** change.

If our C# model and the database are out of sync, the application **will not work correctly**.

A good way to think about migrations is a version history of our database.

![Database version history](assets/images/database-version-history.jpg "Database version history")

Each migration is one step in that history. The workflow always follows the same three steps.

![Model changes workflow](assets/images/model-changes-workflow.jpg "Model changes workflow")

1. We change something in our C# entities.
2. We generate a migration and this is when Entity Framework Core compares our current model with the previous state and generates code describing the difference.
3. We apply that migration, which executes that code against the database and updates its schema.

Let's create our first migration. Open the terminal in our project directory and run the following command:

```bash
    dotnet ef migrations add InitialCreate
```

**InitialCreate** is just the name we are giving this migration. It can be anything, but it should describe what the migration does. 

Entity Framework Core will use it as part of the file name.

This command looks at our C# model and compares it with the previous state, which in this case is empty since this is our first migration.

But how does Entity Framework Core know which classes belong to our model?

Well, it knows the heart of our data definition is the ``DbContext``.

Entity Framework Core looks into it and at the **DbSet** properties defined there. Every entity exposed as a **DbSet** is included in the model and Entity Framework will generate the corresponding table for it.

A new folder called **Migrations** has been added to our project.

Let's see what's inside:

> 20261105042548_InitialCreate.cs       
>       
> LibraryDbContextModelSnapshot.cs

We have two files, one representing the **migration** itself and the other the **snapshot**.

The **migration** file describes a specific change, what to add and what to remove from the database schema.

Each **migration** is a separate file, and together they form the full history of our database changes.

The **snapshot** file is different. It does not describe a change, but it represents the current full state of our model.

Entity Framework Core uses it when generating the next migration. It compares our current C# classes with the snapshot to figure out what has changed. This **snapshot** is always overwritten. It does not grow with the project, it just always reflects the latest state.

When working on a project that lives in source control, those files should be included in the repository alongside the rest of the code.

This way, anyone who clones the repository gets the full history of database changes and can bring their local database to the correct state by simply applying the migrations.

The snapshot should be included as well, since Entity Framework core needs it to generate future migrations correctly.

Neither file contains **secrets**, so there is no reason to keep them out of the repository.

## Migration files

Migrations contains two files.

### Migration

The ``migration.cs`` file

It contains two methods:

``Up()`` is called when we apply the migration. It describes what changes need to be made in the database.

``Down()`` is the exact reverse. It describes how to undo those changes and it is what Entity Framework Core would run if we ever needed to roll back this migration.

Let's look at the ``Up()`` method.

It contains two ``CreateTable()`` calls. One for ``Authors`` and one for ``Categories``, which are the two entities we have defined so far.

For each table, Entity Framework Core has mapped our C# properties to database columns.

#### Authors CreateTable code

```bash
migrationBuilder.CreateTable(
    name: "Authors",
    columns: table => new
    {
        Id = table.Column<int>(type: "integer", nullable: false)
            .Annotation("Npgsql:ValueGenerationStrategy", NpgsqlValueGenerationStrategy.IdentityByDefaultColumn),
        FirstName = table.Column<string>(type: "text", nullable: false),
        LastName = table.Column<string>(type: "text", nullable: false),
        YearOfBirth = table.Column<int>(type: "integer", nullable: true)
    },
    constraints: table =>
    {
        table.PrimaryKey("PK_Authors", x => x.Id);
    });
```

The ``Id`` column is defined as an integer with an **auto-increment** strategy. This means PostgresQL will automatically generate a new value for this column whenever we insert a row, which is exactly what we want for a **primary key**.

``FirstName``, ``LastName``, are mapped to **text** columns and marked as not nullable because those properties are non-nullable in our C# classes.

``YearOfBirth`` is mapped to an **integer**.

Each table also gets a primary key constraint defined on the **Id** column.

One thing that is clearly seen here is that Entity Framework Core is heavily convention-driven.

Rather than requiring us to configure everything explicitly, it makes sensible assumptions based on naming patterns.

For example, a property named Id is automatically recognized as a primary or foreign key. No extra configuration is needed.

Similarly, if a property is **nullable** in C#, EF Core will reflect that in the database schema by allowing **null** values in the corresponding column. And **non-nullable** properties will generate a not-null constraint.

These conventions cover most common scenarios and keep your code clean, but they can always 
be overridden with explicit configuration when you need more control.

The last thing in this file is the down method that simply drops both tables.

### Snapshot

This snapshot describes the same model, but in a different form.

Instead of instructions for changing the database, it is a full description of what our model looks like right now.

In our case, the difference is not big, because we started with an empty database. But later, with our fifth or sixth migration, the difference will be noticeable.

The migration will still only describe the latest change, while the snapshot will describe the entire current state of the database.

We can see each Entity defined with all its properties, their column types, which ones are required and which column serves as the primary key.

#### Author Entity method

```bash
modelBuilder.Entity("LibraryManagement.Api.Entities.Author", b =>
    {
        b.Property<int>("Id")
            .ValueGeneratedOnAdd()
            .HasColumnType("integer");

        NpgsqlPropertyBuilderExtensions.UseIdentityByDefaultColumn(b.Property<int>("Id"));

        b.Property<string>("FirstName")
            .IsRequired()
            .HasColumnType("text");

        b.Property<string>("LastName")
            .IsRequired()
            .HasColumnType("text");

        b.Property<int?>("YearOfBirth")
            .HasColumnType("integer");

        b.HasKey("Id");

        b.ToTable("Authors");
    });
```

The ``ValueGeneratedOnAddCol`` on the ``Id`` property tells Entity Framework Core that this value is generated by the database on **Insert**.

**Note:** this file is auto-generated and should never be edited by hand.

## Applying a Migration

**Note:** make sure you have the container up and running.

Let's now see how to apply a migration to our database. Once again, we will do it in the terminal.

```bash
    dotnet ef database update
```

When we run this command, Entity Framework Core takes the ``Up()`` method from our migration and translates it into SQL.

In our case, ``CreateTable`` statements that actually create the ``Authors`` and ``Categories`` tables in PostgreSQL.

The command has run, but there seems to be an error. Taking a close look at it, we will see it is nothing to worry about. Entity Framework Core tried to access a table called EF Migrations History, but it was not found, which makes sense since our database is brand new.

In this case, EF Core simply creates this table from scratch. If we scroll down, we will see that the migration was applied successfully.

## Browsing the database in DBeaver

**Note:** in DBeaver I connected to the PostgreSQL database. I should have connected to the ``librarydb`` database. Make sure you connect to the right database.

Unfold the view of our database all the way to the Tables directory And here we can see our ``Authors`` and ``Categories`` tables.

Look at ``Authors``. We can see four columns, as expected. This matches exactly what we defined in our C# classes.

Now this only shows the database schema, but what if we want to see the actual data? Well, we can just right-click on the table and select View Data. Currently, nothing is here.

Generally speaking, we will not modify our data manually from DBeaver. It will be our API that manipulates the data.

Let's see how to manipulate the data.

There are two approaches we can take.

One more visual and one with generated SQL.

In DBeaver right-click the ``Authors`` table and select ``View Data``. This will show you the table grid view with no data.

To add an author We can right-click on a row, select Edit, and here we have a couple of options, like adding or deleting a row. Add an ``Author``.

![Adding a row into the database](assets/images/insert-data.jpg "Adding a row into the database")

You will have to add the ``Id`` value.

Save the data by pressing ``Ctrl-S``.

If you know a little SQL and want to see the queries that update the data, you can take the other approach.

Let's right-click on the author's table and select Generate SQL.

There are many templates here, but for now we need Insert.

```bash
INSERT INTO public."Authors"
("Id", "FirstName", "LastName", "YearOfBirth")
VALUES(2, 'Charley', 'Robson', 2011);
```

Let's copy it and open the SQL Editor. We can do it by right-clicking on our database and selecting SQL Editor, New SQL Script.

To refresh this view, we can right-click on the table and select Refresh.

Let's remove those records to leave the database clean. We can right-click on a record, select Edit and Delete Current Row. Don't forget to save. 

For the second author.

```bash
DELETE FROM public."Authors"
WHERE "Id"=2;
```

After refreshing this view, we see that the table is empty once again.

### __EFMigrationsHistory

You may have noticed that there is a third table here, ``__EFMigrationsHistory``. 

Let's see what data is inside. This table is created and managed automatically by Entity Framework Core. It keeps track of which migrations have already been applied to this database.

Every time we run .NET Entity Framework Database Update, Entity Framework Core checks this table first. If a migration is already listed here, it skips it. If it's not, it applies it and adds a record.

This is how Entity Framework Core knows the current state of the database and avoids applying the same migration twice.

## Troubleshooting migrations and database issues

These are a Few straightforward options that should cover most situations you might encounter during development.

The first scenario is when your database gets into a broken state.

This can happen if migrations get out of sync or if you change something manually and you are not sure how to reverse it. The quickest fix is to drop the database entirely and recreate it from scratch.

```bash
    dotnet ef database drop
```

Let's recreate it.

```bash
    dotnet ef database update
```

The first command removed the database completely. The second recreated it by replaying all migrations from the beginning. In a development environment, this is perfectly fine. On a production database, you would never do this.

Dropping a database means losing all data permanently.

The second case is when you want to remove a migration. In most cases, this should not be necessary. Even if you model something incorrectly, the cleaner approach is to fix your entities and create a new migration on top. But if you do need to remove one, here is how it works.

```bash
    dotnet ef migrations remove
```

If the migration has not been applied to the database yet, simply run this command. This removes the most recent migration. If you need to remove more than one, run the command repeatedly until the migration you want to keep is the latest one.

Do not remove migration files manually.

The ``migrations remove`` command also updates the snapshot file and the two always need to be in sync.

Add some random change in entity and then create and apply new migration.

In Categories:

```bash
    public class Category
    {
        public int Id { get; set; }
        public required string Name { get; set; }
        public string Name2 { get; set; }
    }
```

We added a property to the ``Category`` Entity.

Create a new migration.

```bash
    dotnet ef migrations add AddName2ToCategory
```

Now update it.

```bash
    dotnet ef database update
```

So in our case, the migration is applied.

To revert it, we need to revert the database to the last migration we want to keep.

```bash
    dotnet ef database update InitialCreate
```

The database is rolled back. Once it is done, we can remove the migration locally with .NET Entity Framework migrations remove.

```bash
    dotnet ef migrations remove
```

As you can see, the newest migration file is gone.

## Updating CategoriesEndpoints to use the database

Now that we have a database set up, let's connect everything together.

The first thing we can do is remove the ``_categories`` private list that we have been using for data.

This will also cause compilation errors in all the places we need to update, so it will be easier to spot them.

Now, instead of reading from and writing to that list, every method will talk directly to the database, using our library **dbContext**.

To make that possible, we need to add it as a parameter to each method that needs database access.

### GetAll

```bash
private static async Task<Ok<List<Category>>> GetAll(LibraryDbContext dbContext)
{
    var categories = await dbContext.Categories.ToListAsync();
    return TypedResults.Ok(categories);
}
```

We inject the **dbContext** as a parameter. MinimalAPI will resolve it automatically from the dependency injection container.

We now call ``ToListAsync`` on ``dbContext.Categories``, which translates to a select query on the 
database and returns all rows from the Categories table.

Notice that we are using ``await`` here for the first time, and this is exactly the moment it starts to make sense.

Querying a database is an operation that takes time. Our application sends a request to the database and waits for the response. With ``await``, instead of blocking the thread while waiting, we free it up to handle other 
requests in the meantime. This is also why our methods return ``Task``.

They are truly asynchronous now.

### GetById

```bash
private static async Task<Results<Ok<Category>, ProblemHttpResult>> GetById(int id, LibraryDbContext dbContext)
{
    var category = await dbContext.Categories.FindAsync(id);
    if (category == null)
    {
        return TypedResults.Problem(
                detail: $"Category with Id {id} not found.",
                statusCode: StatusCodes.Status404NotFound);
    }

    return TypedResults.Ok(category);
}
```

Now let's change how we retrieve a ``Category`` by ``Id``. ``FindAsync`` looks up a record by its **primary key**. It is the most efficient way to retrieve a single entity by ``Id``.

You might wonder why we are not using ``FirstOrDefaultAsync`` here, since we could. Entity Framework Core supports link operations on ``DbSet``, so this would also work. But ``FindAsync`` is simply a more convenient and slightly more efficient option, specifically for looking up a record by its **primary key**.

For anything more complex than that, for example, finding a ``Category`` by ``name``, we would use ``FirstOrDefaultAsync``.

### Create

```bash
private static async Task<Results<CreatedAtRoute<Category>, ProblemHttpResult>> Create(Category category, LibraryDbContext dbContext)
{
    if (string.IsNullOrEmpty(category.Name))
    {
        return TypedResults.Problem(
                detail: "Category name is required.",
                statusCode: StatusCodes.Status400BadRequest);
    }

    if (await dbContext.Categories.AnyAsync(existingCategory => existingCategory.Name.Equals(category.Name, StringComparison.OrdinalIgnoreCase)))
    {
        return TypedResults.Problem(
                detail: $"A category with the name '{category.Name}' already exists.",
                statusCode: StatusCodes.Status409Conflict);
    }

    dbContext.Categories.Add(category);
    await dbContext.SaveChangesAsync();

    return TypedResults.CreatedAtRoute(category, "GetCategoryById", new { id = category.Id });
}
```

Our endpoint currently accepts a full Category object, including the ``Id`` field but now that the database generates the ``Id`` automatically, it does not make sense for the client to provide it.

The proper solution is to introduce a separate **DTO** for the create request an object that only contains the fields the client should actually send. We will do exactly that soon.

For now, let's keep things simple and just ignore whatever Id the client sends. The database will set its own value if it will be missing.

We no longer need to check for a duplicate Id. The database will handle that. So we will remove this check.

Let's also change how we check whether a Category with the same ``Name`` already exists.

We now use ``AnyAsync`` against the database instead of our local list. It works just the same as the any method from Linq, but it is asynchronous and because we are operating on a dbSet, this will be translated to SQL and executed on the database side so it will not load all categories into memory and filter them locally, but rather filter them on the database side, which is much more efficient.

Now let's change how we actually add the new ``Category``. When we call ``dbContext.Categories.Add``, Entity Framework Core tracks that change in memory, but doesn't immediately write anything to the database.

It is only when we call ``SaveChangesAsync`` that Entity Framework Core translates our pending changes into SQL and executes them against the database.

This means we can make multiple changes and commit them all at once in a single transaction.

Also remember, whenever you make changes to entities, always call ``SaveChangesAsync`` to persist them in the database.

### Update

```bash
private static async Task<Results<Ok<Category>, ProblemHttpResult>> Update(int id, Category updatedCategory, LibraryDbContext dbContext)
{
    if (string.IsNullOrEmpty(updatedCategory.Name))
    {
        return TypedResults.Problem(
                detail: "Category name is required.",
                statusCode: StatusCodes.Status400BadRequest);
    }

    var existingCategory = await dbContext.Categories.FindAsync(id);

    if (existingCategory == null)
    {
        return TypedResults.Problem(
                detail: $"Category with Id {id} not found.",
                statusCode: StatusCodes.Status404NotFound);
    }

    if (await dbContext.Categories.AnyAsync(category => category.Id != id && category.Name == updatedCategory.Name))
    {
        return TypedResults.Problem(
                detail: $"A category with the name '{updatedCategory.Name}' already exists.",
                statusCode: StatusCodes.Status409Conflict);
    }

    existingCategory.Name = updatedCategory.Name;
    await dbContext.SaveChangesAsync();

    return TypedResults.Ok(existingCategory);
}
```

We will use ``FindAsync`` to retrieve the category by ``Id``.

Let's also update the ``DuplicateName`` check to use an ``async``.

After we update the ``Category``'s name, we need to save the changes. Entity Framework Core is tracking this entity, meaning it keeps an internal record of what the object looked like when we first retrieved it.

When we call ``SaveChangesAsync``, it compares the current state with that original snapshot and generates an update SQL statement for only the columns that actually changed.

### Delete

```bash
private static async Task<Results<NoContent, ProblemHttpResult>> Delete(int id, LibraryDbContext dbContext)
{
    var categoryToBeDeleted = await dbContext.Categories.FindAsync(id);

    if (categoryToBeDeleted == null)
    {
        return TypedResults.Problem(
                detail: $"Category with Id {id} not found.",
                statusCode: StatusCodes.Status404NotFound);
    }

    dbContext.Categories.Remove(categoryToBeDeleted);
    await dbContext.SaveChangesAsync();

    return TypedResults.NoContent();
}
```

Once we have confirmed the ``Category`` exists, we remove it from the ``DbSet`` and save the changes.

Let's run the application and test everything in Scalar.

## Updating AuthorsController to use a database

This is very similar to what we did for ``Categories`` endpoints, we will ask AI to do it for us.

**Note:** before I did this I committed all of my previous changes to Git.

### Prompt

> ASP.Net Core 10 Web API:      
>       
> Update AuthorsController to use LibraryDbContext instead of _authors list.        
>       
> Delete the list and follow the same pattern in Handler methods as CategoriesEndpoints keeping in mind that > AuthorsController is a controller not a minimal API

I have uploaded the changes into my ``AuthorsController``.

The one major difference between using ``LibraryDbContext`` in Controllers versus Minimal API is that in Minimal API, the context is passed as a parameter to each individual method.

In a controller, we inject it once through the constructor, and then every method in the class has access to it. This is the standard pattern for dependency injection in Controllers.

In my AI it generated the following code:

```bash
...
private readonly LibraryDbContext _dbContext;

public AuthorsController(LibraryDbContext dbContext)
{
    _dbContext = dbContext;
}
...
```

There is another way to do this by using a **Primary Constructor**.

```bash
...
[ApiController]
[Route("api/[controller]")]
public class AuthorsController(LibraryDbContext dbContext) : ControllerBase
{
    ...
```

I will change my code to this but if you don't like primary constructors, you can always use the plain old constructor with a private field.

### GetById and GetAll

```bash
...
[HttpGet("{id:int}")]
[ProducesResponseType(typeof(Author), StatusCodes.Status200OK)]
[ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
public async Task<ActionResult<Author>> GetById(int id)
{
    var author = await dbContext.Authors.FindAsync(id);

    if (author == null)
    {
        return Problem(
            detail: $"Author with ID {id} not found.",
            statusCode: StatusCodes.Status404NotFound);
    }

    return Ok(author);
}

[HttpGet()]
[ProducesResponseType(typeof(List<Author>), StatusCodes.Status200OK)]
public async Task<ActionResult<List<Author>>> GetAll()
{
    var authors = await dbContext.Authors.ToListAsync();
    return Ok(authors);
}
...
```

### Create

```bash
...
[HttpPost()]
[ProducesResponseType(typeof(Author), StatusCodes.Status201Created)]
[ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status400BadRequest)]
[ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status409Conflict)]
public async Task<ActionResult<Author>> Create(Author author)
{
    if (string.IsNullOrWhiteSpace(author.FirstName) || string.IsNullOrWhiteSpace(author.LastName))
    {
        return Problem(
            detail: "First name and last name are required.",
            statusCode: StatusCodes.Status400BadRequest);
    }

    if (await dbContext.Authors.AnyAsync(existingAuthor => existingAuthor.Id == author.Id))
    {
        return Problem(
            detail: $"An author with ID {author.Id} already exists.",
            statusCode: StatusCodes.Status409Conflict);
    }

    dbContext.Authors.Add(author);
    await dbContext.SaveChangesAsync();

    return CreatedAtAction(nameof(GetById), new { id = author.Id }, author);
}
...
```

This code has a minor issue. Before using the database, the ``Create`` endpoint could produce a **409 Conflict** response, because the ``Id`` was provided by the user, and it could conflict with an existing ``Id`` in this storage. Now, the database assigns ``Id``s, and guarantees they are always unique.

So this response is no longer possible, and this check is no longer needed. Remove them.

### Final code

```bash
...
[HttpPost()]
[ProducesResponseType(typeof(Author), StatusCodes.Status201Created)]
[ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status400BadRequest)]
public async Task<ActionResult<Author>> Create(Author author)
{
    if (string.IsNullOrWhiteSpace(author.FirstName) || string.IsNullOrWhiteSpace(author.LastName))
    {
        return Problem(
            detail: "First name and last name are required.",
            statusCode: StatusCodes.Status400BadRequest);
    }

    dbContext.Authors.Add(author);
    await dbContext.SaveChangesAsync();

    return CreatedAtAction(nameof(GetById), new { id = author.Id }, author);
}
...
```

Remember, conflict can still happen for ``Categories``, because their names must be unique, and if they are not, **409** is the appropriate response.

### Update

Is basically the same.

```bash
...
[HttpPut("{id:int}")]
[ProducesResponseType(typeof(Author), StatusCodes.Status200OK)]
[ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status400BadRequest)]
[ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
public async Task<ActionResult<Author>> Update(int id, Author author)
{
    var existingAuthor = await dbContext.Authors.FindAsync(id);

    if (existingAuthor == null)
    {
        return Problem(
            detail: $"Author with ID {id} not found.",
            statusCode: StatusCodes.Status404NotFound);
    }

    if (string.IsNullOrWhiteSpace(author.FirstName) || string.IsNullOrWhiteSpace(author.LastName))
    {
        return Problem(
            detail: "First name and last name are required.",
            statusCode: StatusCodes.Status400BadRequest);
    }

    existingAuthor.FirstName = author.FirstName;
    existingAuthor.LastName = author.LastName;
    existingAuthor.YearOfBirth = author.YearOfBirth;

    await dbContext.SaveChangesAsync();

    return Ok(existingAuthor);
}
...
```

### Delete

Once again, basically the same.

```bash
...
[HttpDelete("{id:int}")]
[ProducesResponseType(StatusCodes.Status204NoContent)]
[ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
public async Task<ActionResult> Delete(int id)
{
    var authorToBeDeleted = await dbContext.Authors.FindAsync(id);

    if (authorToBeDeleted == null)
    {
        return Problem(
            detail: $"Author with ID {id} not found.",
            statusCode: StatusCodes.Status404NotFound);
    }

    dbContext.Authors.Remove(authorToBeDeleted);
    await dbContext.SaveChangesAsync();

    return NoContent();
}
...
```

Test your endpoints.

### Why do we use a concrete DbContext?

Looking at our controllers, you might notice something that raises a reasonable question. We are injecting a concrete class, ``LibraryDbContext``, not an interface.

This goes against the dependency inversion principle, which says we should depend on abstractions, not concrete implementations. So why are we doing this?

Well, Microsoft simply designed ``DbContext`` this way. There is no built-in IDbContext interface to inject instead and this is not an oversight. It is a deliberate design decision.

A large number of real-world ASP.NET Core applications inject ``DbContext`` directly.

Now, depending on a concrete class rather than an interface usually creates a problem, especially in tests.

Typically, when we want to test a class in isolation, we replace its dependencies with mocks. Fake implementations that we fully control. But to do that, we need an interface that our mock will implement.

If we depend on a concrete class, we can't swap it out for mock. In this case, though, it is not a problem in practice, because Entity Framework Core has a built-in in-memory database provider, which allows us to replace the real PostgreSQL database with a lightweight in-memory one during tests.

This is actually more convenient than manually mocking every database call. And it produces tests that are shorter and easier to read.

### Repository pattern

There is an alternative pattern worth knowing about. Called the ``Repository`` pattern. The idea is to introduce an abstraction layer between your business logic and your data source. So instead of talking to the database directly, your code talks to a repository, which handles all the data access behind the scenes. This makes the data source swappable.

In our case, applying this pattern would mean hiding ``DbContext`` behind an interface. For example, ``IAuthorRepository``, so that the controller depends on an interface rather than concrete ``DbContext`` implementation. This is doable, but we should skip it.

The repository pattern is itself complex, and most real-life ASP.NET APIs do not implement it anyway and maybe most notably, ``DbContext`` itself is already based on the repository pattern internally.

Adding another layer on top of that would be more overhead than it is worth. 

### Summary

Injecting library ``DbContext`` directly is a perfectly valid, widely used approach.

## Dependency Injection in ASP.Net Core

Now that we have updated the ``AuthorsController``, let's take a moment to look at something we introduced without fully explaining it.

There is a question worth asking here.

```bash
    ...
    [ApiController]
    [Route("api/[controller]")]
    public class AuthorsController(LibraryDbContext dbContext) : ControllerBase
    {
        ...
```

Where does the ``LibraryDbContext`` instance actually come from? We never call new ``AuthorsController`` anywhere in our code. 

```bash
    var authorsController = new AuthorsController(...);
```

The controller just gets created somehow, and the ``LibraryDbContext`` is already there, ready to use.

But how? The answer is **Dependency Injection**. This is not a concept specific to .NET. It is a general programming pattern found in practically every modern framework.

ASP.NET Core has its own built-in **Dependency Injection** container, which covers the needs of most applications without any additional libraries.

So what problem does dependency injection actually solve?

Well, without it, our controller would have to create its own ``LibraryDbContext``.

It would look something like this:

```bash
    public AuthorsController()
    {
        _dbContext = new LibraryDbContext(...);
    }
```

But this is problematic. The controller would need to know how to configure the ``dbContext``, the connection string, the database provider, everything. Remember, this is just one controller. We will have many more classes that need ``LibraryDbContext``.

So that configuration logic would be scattered across the entire project.

There is a better solution. We can have a central registry called the **Dependency Injection** container. Think of it as a box with all types our application may need.

![Dependency Injection Container](assets/images/dependency-intection-container.jpg "Dependency Injection Container")

We register our services once, and from this point on, the container knows how to create them. We actually do that already.

Every call we make on ``Builder.Services`` in the ``Program.cs`` class is a registration in that container, like putting a new service into that box, making it ready to use.

Add ``dbContext``, add ``Controllers``, add ``OpenAPI``, all of it goes into the same container. Without that registration, the container doesn't know the service exists and cannot inject it anywhere.

For example, consider this line:

```bash
    builder.Services.AddDbContext<LibraryDbContext>(options =>
    ...
```

From that point on, whenever any class needs a LibraryDbContext, it simply declares that need as a **Constructor Parameter** in a Controller or as a **Method Parameter** in minimal API. And the container simply delivers it. This is where the word **Inject** in dependency injection comes from. The container injects the dependency into the class that needs it.

It is worth remembering that every service we register has a specific lifetime.

## Service lifetime

When we register a service, we also tell the container how long its instance should live.

There are three options.

**Singleton** means one instance for the entire lifetime of the application. Everyone who asks for this service gets the same object. A good example would be an in-memory cache. Something that holds data globally and is meant to be shared across the entire application.

**Transient** means a new instance every time someone asks for it. It is good for lightweight, stateless services. For example, a simple validator that takes some input, checks it, and returns a result.

**Scoped** means we have one instance per HTTP request. Everyone who asks for this service within the same request gets the same instance. When the request ends, the instance is disposed of.

For example, a **DbContext** should be scoped. It tracks changes to entities during a request, so it should live exactly as long as the request.

The built-in services are usually configured upfront with the correct lifetime.

But when we start registering our own services, we will think what lifetime is best for them.

Let's say we create an ``IEmailService`` interface and an ``EmailService`` class that implements it.

```bash
    public interface IEmailService { }

    Public class EmailService : IEmailService { }
```

This is how we would register it in ``Program.cs`` class.

```bash
    builder.Services.AddScoped<IEmailService, EmailService>();
```

This tells the container, whenever someone asks for ``IEmailService``, give them an instance of the ``EmailService`` class. And then in a Controller, we simply declare that we need this dependency.

The controller then depends on the interface, not a concrete class. 

```bash
    public class AuthorsController(LibraryDbContext dbContext, IEmailService emailService) : ControllerBase
```

This means it follows the **Dependency Inversion** principle from solid. This means we can swap the implementation at any time. For example, use a fake EmailService in tests, without touching the controller at all.

This loose coupling between classes is the real power of dependency injection. We don't have any custom services in our project yet, but we will. And when we get there, this is exactly the pattern we will follow.

## Adding seed data

When we test our API as developers, it is convenient to already have some data in the database.

A few Categories, Books, Authors, and so on. This gives us a foundation we can use to run any tests we want.

We could add this data manually, through the API, or directly in the DBeaver but that has a couple of downsides. First, if we drop and recreate the database for any reason, we lose all of it and have to set it up again from scratch. Second, if a new developer joins the project, they would have to do the same thing, rather than getting a ready-to-use database from the start.

The solution is **seed data**, a set of initial records that are automatically added to the database when the application starts. We configure it once in our code, and from that point on, even if the database is dropped 
and recreated, the seed data will be there every time the app starts.

This will be a **development-only** feature. Let's see how this works.

Create a new static class named ``DataSeeder`` in the ``DataAccess`` folder. We make it a static class, because it doesn't hold any state. It is just a container for a method we will call once at startup.

```bash
public static async Task SeedAsync(IServiceProvider serviceProvider)
{
    using var scope = serviceProvider.CreateScope();
    var dbContext = scope.ServiceProvider.GetRequiredService<LibraryDbContext>();

    if (!await dbContext.Authors.AnyAsync())
    {
        var authors = new List<Author>
        {
            new Author { FirstName = "George", LastName = "Orwell", YearOfBirth = 1903 },
            new Author { FirstName = "Jane", LastName = "Austen", YearOfBirth = 1775 },
            new Author { FirstName = "Mark", LastName = "Twain", YearOfBirth = 1835 },
            new Author { FirstName = "J.R.R.", LastName = "Tolkien", YearOfBirth = 1892 },
            new Author { FirstName = "Ernest", LastName = "Hemingway", YearOfBirth = 1899 }
        };

        await dbContext.Authors.AddRangeAsync(authors);
        await dbContext.SaveChangesAsync();

        var categories = new List<Category>
        {
            new Category { Name = "Fiction" },
            new Category { Name = "Non-Fiction" },
            new Category { Name = "Science Fiction" },
            new Category { Name = "Fantasy" },
            new Category { Name = "Biography" }
        };

        await dbContext.Categories.AddRangeAsync(categories);
        await dbContext.SaveChangesAsync();
    }
}        
```

Add the ``SeedAsync`` method. As a parameter, we pass an ``IServiceProvider``.

To understand what it is and why we need it, we need to think about when this method will be called. It will run at application startup, not within any particular HTTP request and since this method writes to the database, it needs a library ``dbContext`` but as we just learned, ``dbContext`` is a scoped service. It is created at the start of an HTTP request and disposed of at the end.

But when we call ``SeedAsync``, there is no **HTTP request**. So we need another way to get ``dbContext``. 
That is exactly what ``IServiceProvider`` is for. It can create a scope manually, even outside of an HTTP request.

We create a scope manually and use it to resolve library dbContext.

The **using** keyword ensures the scope and everything it created, including the ``dbContext``, is properly disposed when we are done.

Now let's seed some ``Authors``. Before adding anything, we check whether there are already ``authors`` in the database. If this returns true, meaning there are no ``Authors``, we will add some.

If we ever need a clean slate, we can just drop the database and restart.

Now let's add the ``Authors``. Write a comment describing what I need, and Copilot suggests a list.

> // Add a list of 5 well known authors with their FirstName and LastName and YearOfBirth

This will generate them for you. Let's save them to the database. We used ``AddRangeAsync`` instead of calling ``Add`` multiple times. It is a more convenient way to add multiple records at once.

Now let's do the same for ``Categories``.

Finally we wire this up in ``Program.cs``. We only want to call ``SeedAsync`` when running in the development environment.

Delete all existing ``Authors`` and ``Categories`` so we start with a clean slate.

Run the application and this will seed the database.

Every time we start fresh with an empty database, the seed data will be there waiting for us and since this all happens inside the seeder, which is part of our codebase, any developer who joins the team will get this behavior automatically the first time they set up and run the project.

## Switching to Microsoft SQL Server

There are 3 things to change.

* The packages.
* 1 line in ``Program.cs``
* The connection string. 

Let's start with packages.

First, we remove the PostgreSQL provider:

```bash
    dotnet remove Npgsql.EntityFrameworkCore.PostgreSQL
```

Install the SQL Server provider:

```bash
    dotnet add package Microsoft.EntityFrameworkCore.SQLServer
```

Next, ``Program.cs``.

This is the only line that changes here:

```bash
builder.Services.AddDbContext<LibraryDbContext>(options =>
{
    options.UseSQLServer(builder.Configuration.GetConnectionString("LibraryDb"));
});
```

And now we need to update the connection string in ``appsettings.development.json``.

```bash
"ConnectionStrings": {
  "LibraryDb": "Server=localhost,11433;Database=librarydb;Trusted_Connection=True;"
}
```

The trusted connection equals true means we are using Windows authentication.

There is one more thing we need to address. Our existing migrations were generated for PostgreSQL.

They contain PostgreSQL-specific code and will not work correctly with SQL Server. We need to **delete** the migrations folder and generate fresh ones.

> dotnet ef migrations add InitialCreate

Now apply the migrations:

> dotnet ef database update

Check see if the tables have been added. If successful they should hae your seed data.

We now have the same API and the same code, but running on a completely different database.

The only things we changed were two NuGet packages, one line in program.cs and the connection stream.

You could now revert all the changes back to PostgreSQL by using your version control.
