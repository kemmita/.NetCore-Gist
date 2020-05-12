1. there are times in our app where we want to create a custom type and use it within one of our domain models, without hvaing to register 
iit as its own table/dbSet. Below is the custom type.
```cs
public class AppUserLocation
    {
        public string Country { get; set; }
        public string City { get; set; }
    }
```
2. We want to use this type in our AppUser Domain model
```cs
public class AppUser : IdentityUser
    {
        public string Title  { get; set; }
        public string Bio { get; set; }
        public AppUserLocation Location { get; set; }
    }
```
3. In order to do this, we will need to register AppUserLocation as an owned type of AppUser
```cs
public class ApplicationDbContext : IdentityDbContext<AppUser>
    {
        public ApplicationDbContext(DbContextOptions options) : base(options)
        {

        }
        public DbSet<AppUser> AppUsers { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);

            modelBuilder.Entity<AppUser>()
                .OwnsOne(typeof(AppUserLocation), "Location");
        }
    }
```
4. Add a migration and then update your db, you will see in your AppUser table in the db, that you have added two new columns
location_city and location_country. This keeps us from needing to create a new table altogether.
