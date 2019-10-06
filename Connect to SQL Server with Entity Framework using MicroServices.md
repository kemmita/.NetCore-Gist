1. We have 4 projects all together, we have an API project that was built with the .NET Core WEBAPI selection. 
2. We have an .NET Core Class Lib that will be named Application, this application will need to be referenced by the API
3. We have an .NET Core Class Lib that will be named Persistence, this project will communicated with the DB, Application class lib will need to reference this class lib.
4. Lastly we have a .net core class lib titled Domain, this will be responisble for holding our domain entities/DB Tables. Persistence lib will need to reference the Domain lib
5. To get this working we will not need to touch the Application class lib.
6. First, in the Domain class lib, create a new model, "Value"
```cs
namespace Domain
{
    public class Value
    {
        public int Id { get; set; }
        public string Name { get; set; }
    }
}

```
7. Next to go to the Persistence Lib and download two packages to the lib Entity Framework Core and Entity Framework Core SQL Server. Make sure they are the same version 2.4 for example.
8. Now create your new class ApplicationDbContext to gain access to entity framework
```cs
//notice we are using the Domain Class Lib so that we can gain access to the Value Model/Entity
using Domain;
using Microsoft.EntityFrameworkCore;

namespace Persistence
{
    public class ApplicationDbContext: DbContext
    {
        public ApplicationDbContext(DbContextOptions options): base(options)
        {

        }
        public DbSet<Value> Values { get; set; }
    }
}

```
9. we will start in the API in the Startup.cs file.
```cs
       public void ConfigureServices(IServiceCollection services)
        {
        //If we did not refernce the Application class lib which in turn references the persistence class lib
        //we would not have access to the DB context below.
            services.AddDbContext<ApplicationDbContext>(opt =>
                {
                    opt.UseSqlServer(Configuration.GetConnectionString("DefaultConnection"));
                });
            services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_2);
        }
```
10. Next go into appsettings.json
```json
  "ConnectionStrings": {
    "DefaultConnection": "Server=DESKTOP-86JBBTP;Database=React_Core;Integrated Security=True; Trusted_Connection=Yes;"
  },
```
11. Lastly go into Program.cs, in the main method, we will set the migration to run if the database has not yet been created. 
```cs
        public static void Main(string[] args)
        {
            var host = CreateWebHostBuilder(args).Build();
            using (var scope = host.Services.CreateScope())
            {
                var services = scope.ServiceProvider;
                try
                {
                    var context = services.GetRequiredService<ApplicationDbContext>();
                    context.Database.Migrate();
                }
                catch (Exception e)
                {
                    var logger = services.GetRequiredService<ILogger<Program>>();
                    logger.LogError(e, "An error has occured during Database Scaffolding!");
                }
            }
            host.Run();
        }
```
12. Run the command below to run the first migration and specify the persistence layer and the API
```
add-migration initial -p Persistence -s API
```
13. Run the application and you should see a new db "Database=React_Core" in ssms.
