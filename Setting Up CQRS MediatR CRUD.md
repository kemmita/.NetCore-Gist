1. First install "MediatR.Extensions.Microsoft.DependencyInjection" from nugget to your Application library.
2. Next go to your API directory and open your Startup class to inject MediatR.
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
            //we will have lots of handlers, more than just "ActivitiesList" but we need only do this once.
            services.AddMediatR(typeof(ActivitiesList.Handler).Assembly);
            services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_2);
        }

```
3. READ ALL! In the Application library we will want to create a new directory for each different entity type. We will be working with Activities, so we will create an activities directory. Inside of the new directory create a class describing what it should return, in my case, "ActivitiesList."
```cs
   public class ActivitiesList
    {
        //below we define our query, our query should return a list of activities.
        public class Query : IRequest<List<Activity>> { }

        //we then create a class Handler that will take in a query that was defined above and return a type of List<Activity> 
        public class Handler : IRequestHandler<Query, List<Activity>>
        {
            //below we inject our db context
            private readonly ApplicationDbContext _db;
            public Handler(ApplicationDbContext db)
            {
                _db = db;
            }
            //We then finally execute our task Handle. Fetching an entire list of elements with no arguments will allow us to ignore
            //the params for now. We simply fetch the data from the db.
            public async Task<List<Activity>> Handle(Query request, CancellationToken cancellationToken)
            {
                var activities = await _db.Activities.ToListAsync();
                return activities;
            }
        }
    }
```
4. Now in our controller we will need to send a new query of type ActitivitiesList to our Application Lib to our Handle method we defined above.
```cs
namespace API.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class ActivitiesController : ControllerBase
    {
        private readonly IMediator _mediator;
        public ActivitiesController(IMediator mediator)
        {
            _mediator = mediator;
        }

        [HttpGet]
        public async Task<ActionResult<List<Activity>>> GetListOfAllActivities()
        {
            return await _mediator.Send(new ActivitiesList.Query());
        }
    }
}
```
