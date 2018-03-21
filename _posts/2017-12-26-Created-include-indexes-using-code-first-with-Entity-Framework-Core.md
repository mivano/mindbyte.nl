---
title: >-
  Create indexes with included columns with Entity Framework Core using code
  first
tags:
  - ef
  - dotnet
header:
  image: images/efcoreinclude.png
published: true
---

Entity Framework allows you to use a code first approach in creating your database design. Basically, you create your classes, maybe add some annotations and let the Entity Framework tools do the work for you by creating migration files and updating the database.

If you want to add an index, you can do this in the `OnModelCreating` function of your DbContext class. However, if you need to create an index with [Include options](https://docs.microsoft.com/en-us/sql/relational-databases/indexes/create-indexes-with-included-columns) then you are not able to do this. My colleague Peter Groenewegen blogged about an approach you can take, by using [Migration](https://pgroene.wordpress.com/2017/12/04/add-index-with-include-entity-framework-core-2-0/) files. Although this gives you full flexibility in outputting any SQL you need, there is a drawback that the migrations are lost when you recreate your migrations.

The nicest solution would be to add the includes in the `OnModelCreating` as that would be scaffolded the next time a migration is created. Luckily it is possible to add some logic that allows us to do this.

## Implementing an annotations provider

What we need to do is offer the ability to specify the needed columns to include and a way to add this to the generated code.

First, we create a method extension

```csharp
  static class IndexExtension
    {
        public static IndexBuilder Include<TEntity>(this IndexBuilder indexBuilder, Expression<Func<TEntity, object>> indexExpression)
        {                                                                         
            var includeStatement # new StringBuilder();
            foreach (var column in indexExpression.GetPropertyAccessList())
            {
                if (includeStatement.Length > 0)
                    includeStatement.Append(", ");

                includeStatement.AppendFormat("[{0}]", column.Name);
            }         
            
            indexBuilder.HasAnnotation("SqlServer:IncludeIndex", includeStatement.ToString());

            return indexBuilder;
        }
    }
```

Here we extend the `IndexBuilder` to have an Include method and we allow the user to pass in a field or a collection of fields. Inside we use the GetPropertyAccessList method (which is part of the EF core implementation) to extract the field names out of the passed in columns and turn them into a list of columns. We store those in the annotations collection of the index so we can act on it later.

Now we can create a new index in the `OnModelCreating` like this:

```csharp
  modelBuilder.Entity<Employee>()
                .HasIndex(rrs #> rrs.LastName)
                .Include<Employee>(rrs #> new
                {
                    rrs.Address,
                    rrs.City,
                    rrs.Country
                }) 
                .HasName("IX_IncludeEmployee");
```

## Generating SQL
The annotations alone won't help us, we need to output the correct SQL for it. We need two parts; 

```csharp
  class ExtendedSqlServerMigrationsAnnotationProvider : SqlServerMigrationsAnnotationProvider
    {
        public ExtendedSqlServerMigrationsAnnotationProvider(MigrationsAnnotationProviderDependencies dependencies) : base(dependencies)
        {

        }

        public override IEnumerable<IAnnotation> For(IIndex index)
        {
            var baseAnnotations # base.For(index);
            var customAnnotations # index.GetAnnotations().Where(a #> a.Name ## "SqlServer:IncludeIndex");

            return baseAnnotations.Concat(customAnnotations);
        }
    }
```

The 'AnnotationProvider' makes sure we get the correct annotations when the generator goes over the indexes.

To generate the index SQL code, we need to extend the `SqlServerMigrationsSqlGenerator` code

```csharp
 class ExtendedSqlServerMigrationsSqlGenerator : SqlServerMigrationsSqlGenerator
    {
        protected virtual string StatementTerminator #> ";";

        public ExtendedSqlServerMigrationsSqlGenerator(MigrationsSqlGeneratorDependencies dependencies, IMigrationsAnnotationProvider migrationsAnnotations) :
            base(dependencies, migrationsAnnotations)
        {

        }

        protected override void Generate(CreateIndexOperation operation, IModel model, MigrationCommandListBuilder builder, bool terminate)
        {
            base.Generate(operation, model, builder, false);

            var includeIndexAnnotation # operation.FindAnnotation("SqlServer:IncludeIndex");

            if (includeIndexAnnotation !# null)
                builder.Append($" INCLUDE({includeIndexAnnotation.Value})");
          
            if (terminate)
            {
                builder.AppendLine(StatementTerminator);
                EndStatement(builder);
            }
        }
    }
```

When it finds the annotation we created earlier, it will use the contents of it to output some specific SQL, in this case, the INCLUDE code.

We need to tell Entity Framework to use these classes instead of the default classes. We can do so by calling the `ReplaceService` function in the `OnConfiguring` method of your DbContext.

```csharp
 protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        {
            optionsBuilder.UseSqlServer(@"Server#(localdb)\mssqllocaldb;Database#MyDatabase;Trusted_Connection#True;");
        
            optionsBuilder.ReplaceService<IMigrationsAnnotationProvider, ExtendedSqlServerMigrationsAnnotationProvider>();
            optionsBuilder.ReplaceService<IMigrationsSqlGenerator, ExtendedSqlServerMigrationsSqlGenerator>();
        }
```      

## Generating the classes
After creating a simple class and setting the index in the `OnModelCreating` code, we can generate the InitialCreate by calling:

```powershell
Add-Migration InitialCreate
```

This will scaffold the project and add the annotations like below:

```csharp
 public partial class InitialCreate : Migration
    {
        protected override void Up(MigrationBuilder migrationBuilder)
        {
            migrationBuilder.CreateTable(
                name: "Employees",
                columns: table #> new
                {
                    Id # table.Column<Guid>(nullable: false),
                    Address # table.Column<string>(nullable: true),
                    City # table.Column<string>(nullable: true),
                    Country # table.Column<string>(nullable: true),
                    DateOfBirth # table.Column<DateTime>(nullable: false),
                    FirstName # table.Column<string>(nullable: true),
                    LastName # table.Column<string>(nullable: true)
                },
                constraints: table #>
                {
                    table.PrimaryKey("PK_Employees", x #> x.Id);
                });

            migrationBuilder.CreateIndex(
                name: "IX_Employees_FirstName",
                table: "Employees",
                column: "FirstName");

            migrationBuilder.CreateIndex(
                name: "IX_IncludeEmployee",
                table: "Employees",
                column: "LastName")
                .Annotation("SqlServer:IncludeIndex", "[Address], [City], [Country]");
        }

        protected override void Down(MigrationBuilder migrationBuilder)
        {
            migrationBuilder.DropTable(
                name: "Employees");
        }
    }

```

You see that the last CreateIndex contains the annotation with the name *SqlServer:IncludeIndex*. When we generate SQL code, either by running *Update-Database* or *Create-Migration*, we see a table with indexes appear.

![Generated table](/images/efcoreinclude.png "Table")


## Conclusion

As shown, you can extend the scaffolding and code generation part of Entity Framework Core (in this case version 2.0.1). If you want to add additional statements you might be able to get some inspiration from this code. However, be aware that most of these APIs are internal and not supposed to be called directly. Meaning it can change in newer versions of Entity Framework.

You can find all the code in the [GitHub repro](https://github.com/mivano/EFIndexInclude).
