1. First download this nugget package
```
Microsoft.AspNetCore.Identity.EntityFrameworkCore
```
2. Create a new Model in your domain project, AppUser and extend IdentityUser. This will give us all of our default user props we will
need such as password, username etc. If we need to an additional prop, we can do so as we normally would. If we did not want to 
add any additional props, are model would look like below
```cs
using Microsoft.AspNetCore.Identity;

namespace Domain
{
    public class AppUser : IdentityUser
    {
        public string DisplayName { get; set; }      
    }
}
```
2.1 Create a UserObject in the Application Project so that we do not return all 10+ props returned when fetching the AppUser from the db.
```cs
   public class User
    {
        public string DisplayName { get; set; }
        public string Token { get; set; }
        public string Username { get; set; }
        public string Image { get; set; }
    }
```
3. In our persistence proeject, we will need to alter our ApplicationDbContext file a bit. Instead of inheriting from DbContext, we will
inherit from IdentityDbContext<AppUser> ensure to include the name of the user model, the one we craetd above.
```cs
public class ApplicationDbContext: IdentityDbContext<AppUser>
    {
        public ApplicationDbContext(DbContextOptions options): base(options)
        {

        }

        protected override void OnModelCreating(ModelBuilder builder)
        {
            //this is needed for everything to work!
            base.OnModelCreating(builder);
        }
    }
```
4. Then run your migration
```cs
add-migration AddIdentity -p Persistence -s API
update-database // may be uper case, I don't fucking know!
```
5. In our Startup class make the following addtions
```cs
    public void ConfigureServices(IServiceCollection services)
        {
            services.AddDbContext<ApplicationDbContext>(opt =>
                {
                    opt.UseSqlServer(Configuration.GetConnectionString("DefaultConnection"));
                });
            services.AddCors(opt =>
            {
                opt.AddPolicy("CorsPolicy", policy =>
                    {
                        policy.AllowAnyHeader().AllowAnyMethod().WithOrigins("http://localhost:3000");
                    });
            });
            services.AddMediatR(typeof(ActivitiesList.Handler).Assembly);
            services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_2).AddFluentValidation(cfg =>
                {
                    cfg.RegisterValidatorsFromAssemblyContaining<CreateActivity>();
                });
                
            //Everything under here is identity related   
            var builder = services.AddIdentityCore<AppUser>();
            var identityBuilder = new IdentityBuilder(builder.UserType, builder.Services);
            identityBuilder.AddEntityFrameworkStores<ApplicationDbContext>();
            identityBuilder.AddSignInManager<SignInManager<AppUser>>();
            services.AddAuthentication();
        }
```
6. We will now seed our database with test users. This will be done in the persistence project, file Seed.cs 
```cs
    public class Seed
    {
        public static async Task SeedData(ApplicationDbContext dbContext, UserManager<AppUser> userManager)
        {
            if (!userManager.Users.Any())
            {
                var users = new List<AppUser>
                {
                    new AppUser
                    {
                        DisplayName = "Bob",
                        UserName = "bob",
                        Email = "bob@test.com"
                    },
                    new AppUser
                    {
                        DisplayName = "Tom",
                        UserName = "tom",
                        Email = "tom@test.com"
                    },
                    new AppUser
                    {
                        DisplayName = "Jane",
                        UserName = "jane",
                        Email = "jane@test.com"
                    },
                };
                foreach (var user in users)
                {
                    await userManager.CreateAsync(user, "Password@1");
                }
            }
    }
```
7. Now in  our Programm class in the main method we will need to update the static method call to take in an instance of UserManager
and since the static method is async and the main method of the progrma is not, we will append the "Wait()" to the call
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
                    var userManager = services.GetRequiredService<UserManager<AppUser>>();
                    context.Database.Migrate();
                    Seed.SeedData(context, userManager).Wait();
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
8. Add a new "project/class library", "Infrastructure" this lib will be responsible for handling JWT related jobs.
9. 
