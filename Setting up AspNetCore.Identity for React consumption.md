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
3. In our persistence lib, we will need to alter our ApplicationDbContext file a bit. Instead of inheriting from DbContext, we will
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
        }
```
6. We will now seed our database with test users. 
```cs

```
