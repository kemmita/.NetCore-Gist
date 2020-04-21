1. First create your view
```sql
create view DoctorResidence 
	as
	select d.Name, h.HosptialName from Doctors d
	join Hospitals h on h.Id=d.HospitalId
	go
```
2. Next create a model that will mirrior what is returned from the view
```cs
public class DoctorResidence
    {
        public string HosptialName { get; set; }
        public string Name { get; set; }
    }
```
3. Next in your applicationdbcontext, we will configure the two
```cs
 public class ApplicationDbContext : DbContext
    {
        public ApplicationDbContext(DbContextOptions options) : base(options)
        {

        }
        public DbSet<Doctor> Doctors { get; set; }
        public DbSet<Hospital> Hospitals { get; set; }
        public DbSet<DoctorResidence> DoctorResidences { get; set; }
        
        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            // HasNoKey method is needed unless an error will be thrown due to us not having an id prop in our model above
            modelBuilder.Entity<DoctorResidence>()
                .HasNoKey()
                .ToView("DoctorResidence");

        }
    }
```
4. Below is how you call it 
```cs
[HttpGet]
        public async Task<ActionResult<List<DoctorResidence>>> Get()
        {
            return await _db.DoctorResidences.ToListAsync();
        }
```
