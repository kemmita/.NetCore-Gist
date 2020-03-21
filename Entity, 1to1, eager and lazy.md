1. First we will install our needed packages for .NET Core 3.1. needed for 1to1 Eager Load
```
Microsoft.EntityFrameworkCore
Microsoft.EntityFrameworkCore.SqlServer
AutoMapper.Extensions.Microsoft.DependencyInjection
Microsoft.AspNetCore.Mvc.NewtonsoftJson
```
2. Below is our Startup.cs class
```cs
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddAutoMapper(typeof(WeatherForecast).Assembly);
            services.AddDbContext<ApplicationDbContext>(opt =>
            {
                opt.UseSqlServer(Configuration.GetConnectionString("DefaultConnection"));
            });
            services.AddControllers().AddNewtonsoftJson(options =>
                options.SerializerSettings.ReferenceLoopHandling = Newtonsoft.Json.ReferenceLoopHandling.Ignore
            );
        }
```
3. In our domain area, we will create our two domain models. Each user should have one credit card/UserPay and each UserPay should 
have one user
```cs
    public class User
    {
        public int Id { get; set; }
        public string Username { get; set; }
        public UserPay Payment { get; set; }
    }
    
    
     public class UserPay
    {
        public int Id { get; set; }
        public string CreditCardNumber { get; set; }
        public User User { get; set; }
    }
```
4. Below in our persistence area, we have our db context class
```cs
  public class ApplicationDbContext : DbContext
    {
        public ApplicationDbContext(DbContextOptions options) : base(options)
        {

        }
        public DbSet<User> Users { get; set; }
        public DbSet<UserPay> UsersPay { get; set; }
        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            //below we define our 1to1 relationship
            modelBuilder.Entity<User>()
                .HasOne(u => u.Payment)
                .WithOne(up => up.User)
                .HasForeignKey<UserPay>(up => up.Id);
            
            //This will create two tables
        }
    }
```
5. In our application layer, create a UserPayInfoDto. If one user will have one creditcard number as defdined above, then we want to
return just the username and credit card numver to the user.
```cs
    public class UserPayInfoDto
    {
        public string Username { get; set; }
        public string CreditCardNumber { get; set; }
    }
```
6. Add yor migrations and run update db
```
add-migration "one" -p Persistence -s API
//if that command throws an error, run command
Install-Package Microsoft.EntityFrameworkCore.Tools -Version 3.1.2
//check to see if that migration is taken, if it is run command
Update-Database
//If the migration did not stage, run
add-migration "one" -p Persistence -s API
Update-Database
```
7. In our application layer, create a mapping profile
```
    public class MappingProfile : Profile
    {
        public MappingProfile()
        {
            //As long as the props you want to map in the object have the same name and same data type, then this will suffice
            CreateMap<User, UserPayInfoDto>()
                .ForMember(dest => dest.Username, opt => opt.MapFrom(x => x.Username))
                .ForMember(dest => dest.CreditCardNumber, opt => opt.MapFrom(x => x.Payment.CreditCardNumber));

        }
    }
```
8. If we did not create the mapping profile, we would have recieved and object of UserPay with our User Type as defined in the models
Using AutoMapping we can eager load both User and UserPayment and then use a Dto to return only what we want the user to see.
Below is the call to the db.
```cs

        [HttpGet]
        public async Task<ActionResult<UserPayInfoDto>> Get()
        {
           //we use the include method to return the UserPay table data associated with our user
            var user = await _db.Users.Include(x => x.Payment).FirstAsync(x => x.Username == "Jane");

            return _mapper.Map<User, UserPayInfoDto>(user);
        }
```
9. Now we will look at using Lazy Loading lets install the packages we will need
```
Microsoft.EntityFrameworkCore.Proxies
```
10. We will need to modify our startup.cs file abit
```cs
 public void ConfigureServices(IServiceCollection services)
        {
            services.AddAutoMapper(typeof(WeatherForecast).Assembly);
            services.AddDbContext<ApplicationDbContext>(opt =>
            {
                opt.UseSqlServer(Configuration.GetConnectionString("DefaultConnection")).UseLazyLoadingProxies();
            });
            services.AddControllers().AddNewtonsoftJson(options =>
                options.SerializerSettings.ReferenceLoopHandling = Newtonsoft.Json.ReferenceLoopHandling.Ignore
            );
        }
```
11. Now update our applicationDbContext
```cs
  protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        {
            optionsBuilder.UseLazyLoadingProxies();
        }
```
12. Now update our models to include the virtual keyword
```cs
 public class User
    {
        public int Id { get; set; }
        public string Username { get; set; }
        public virtual UserPay Payment { get; set; }
    }
    
      public class UserPay
    {
        public int Id { get; set; }
        public string CreditCardNumber { get; set; }
        public virtual User User { get; set; }
    }
```
13. Now we no longer need to use the inlcude key word! The potential downfall of this, is that our related payment prop will always be returned 
```cs
[HttpGet]
        public async Task<ActionResult<UserPayInfoDto>> Get()
        {
            var user = await _db.Users.FirstAsync(x => x.Username == "Jane");

            return _mapper.Map<User, UserPayInfoDto>(user);
        }
```
