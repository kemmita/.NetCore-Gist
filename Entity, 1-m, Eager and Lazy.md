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
3. In our domain area, we will create our two domain models. Each Hospital should have many Doctors
```cs
    public class Hospital
    {
        public int Id { get; set; }
        public string HosptialName { get; set; }
        public ICollection<Doctor> Doctors { get; set; }
    }
    
        public class D2
    {
        public int Id { get; set; }
        public string Name { get; set; }
    }
```
4. Below in our persistence area, we have our db context class
```cs
    public class ApplicationDbContext : DbContext
    {
        public ApplicationDbContext(DbContextOptions options) : base(options)
        {

        }
        public DbSet<Doctor> Doctors { get; set; }
        public DbSet<Hospital> Hospitals { get; set; }
    }
```
5. Add yor migrations and run update db
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
6. We will now see how to query the data from the db together at once. This will return all hospitals and a list of doctors for
each hospital.
```cs
   [HttpGet]
        public async Task<ActionResult<List<H2>>> Get()
        {
            var hospital = await _db.Hospital.Include(x => x.Doctors).ToListAsync();

            return hospital;
        }
```
7. Now we will use Lazy Loading. First install our needed package.
```
Microsoft.EntityFrameworkCore.Proxies
```
8. Modify startup.cs
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
9. Now update application db context
```cs
 protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        {
            optionsBuilder.UseLazyLoadingProxies();
        }
```
10. Now update our models to include the virtual keyword
```cs
    public class Hospital
    {
        public int Id { get; set; }
        public string HosptialName { get; set; }
        public virtual ICollection<Doctor> Doctors { get; set; }
    }
    
        public class Doctor
    {
        public int Id { get; set; }
        public string Name { get; set; }
    }
```
11.This will load all hospital info along with the docsa for each hospital with the inlcude keyword.
```cs

        [HttpGet]
        public async Task<ActionResult<List<H2>>> Get()
        {
            var hospital = await _db.H2s.ToListAsync();

            return hospital;
        }
```
