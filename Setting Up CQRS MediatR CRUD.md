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
5. READ ONE! 
```cs
public class ActivityDetail
    {
        //specify what param is expected to be passed for this Query
        public class Query : IRequest<Activity>
        {
            public Guid Id { get; set; }
        }

        public class Handler : IRequestHandler<Query, Activity>
        {
            private readonly ApplicationDbContext _db;

            public Handler(ApplicationDbContext db)
            {
                _db = db;
            }
            //using Query defined above wih a param Id of type Guid, we will locate a single activity in the db associated with the Guid id.
            public async Task<Activity> Handle(Query request, CancellationToken cancellationToken)
            {
                return await _db.Activities.FindAsync(request.Id);
            }
        }
    }
```
6. Lastly, in our controller create a new action and send a Query to the handle method defined above.
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
        
        [HttpGet("{id}")]
        public async Task<ActionResult<Activity>> GetActivityDetails(Guid id)
        {
            return await _mediator.Send(new ActivityDetail.Query{Id = id});
        }
    }
}
```
7. **CREATE** Below we will create a new activity. Create a new class in your Application/Activities direcotry titled "CreateActivity"
```cs
 public class CreateActivity
    {
        //this time we have a command instead of a query and we must specify the values that will be passed in the command
        public class Command : IRequest
        {
            public Guid Id { get; set; }
            public string Title { get; set; }
            public string Description { get; set; }
            public string Category { get; set; }
            public string City { get; set; }
            public string Venue { get; set; }
            public DateTime Date { get; set; }
        }

        public class Handler : IRequestHandler<Command>
        {
            private readonly ApplicationDbContext _db;
            public Handler(ApplicationDbContext db)
            {
                _db = db;
            }
            //since we are writing a command, a "POST" we will not be returning data, but storing data.
            //so this time our task will return a type of unit, which is simply an int
            public async Task<Unit> Handle(Command request, CancellationToken cancellationToken)
            {
                var activity = new Activity
                {
                    Id = request.Id,
                    Title = request.Title,
                    Description = request.Description,
                    Category = request.Category,
                    City = request.City,
                    Venue = request.Venue,
                    Date = request.Date,
                };

                _db.Activities.Add(activity);
                
                //to ensure that the data was stored correctly, check that unit is greater than 0
                var success = await _db.SaveChangesAsync() > 0;

                if (success)
                {
                    return Unit.Value;
                }

                throw new Exception("Problem saving new activity");
            }
        }
    }
```
8. Controller
```cs
 [Route("api/[controller]")]
    [ApiController]
    public class ActivitiesController : ControllerBase
    {
        private readonly IMediator _mediator;
        public ActivitiesController(IMediator mediator)
        {
            _mediator = mediator;
        }

         //again we return a unit. notice that we are not specifing a type of Activity in the param field, but a 
         //CreateActivity.Command which we assigned the value of type Activity in the class above.
        [HttpPost("")]
        public async Task<ActionResult<Unit>> Create(CreateActivity.Command command)
        {
            return await _mediator.Send(command);
        }
    }
```
9. **UPDATE** Create a new class in your Application/Activities direcotry titled "CreateActivity"
```cs
 public class EditActivity
    {
        /define the values that will be passed in
        public class Command : IRequest
        {
            public Guid Id { get; set; }
            public string Title { get; set; }
            public string Description { get; set; }
            public string Category { get; set; }
            public string City { get; set; }
            public string Venue { get; set; }
            public DateTime? Date { get; set; }
        }

        public class Handler : IRequestHandler<Command>
        {
            private readonly ApplicationDbContext _db;
            public Handler(ApplicationDbContext db)
            {
                _db = db;
            }
            public async Task<Unit> Handle(Command request, CancellationToken cancellationToken)
            {
                var activity = await _db.Activities.FindAsync(request.Id);

                if (activity == null)
                {
                    throw new Exception("Could not locate that activity");
                }
                else
                {
                    activity.Title = request.Title ?? activity.Title;
                    activity.Description = request.Description ?? activity.Description;
                    activity.Category = request.Category ?? activity.Category;
                    activity.City = request.City ?? activity.City;
                    activity.Venue = request.Venue ?? activity.Venue;
                    activity.Date = request.Date ?? activity.Date;

                    var success = await _db.SaveChangesAsync() > 0;
                    if (success)
                    {
                        return Unit.Value;
                    }
                    throw new Exception("Problem saving changes");
                }

            }
        }
    }
```
10. Controller
```cs
   [Route("api/[controller]")]
    [ApiController]
    public class ActivitiesController : ControllerBase
    {
        private readonly IMediator _mediator;
        public ActivitiesController(IMediator mediator)
        {
            _mediator = mediator;
        }
        
        //below we see how to pass in the id needed to update a specific activity 
        [HttpPut("{id}")]
        public async Task<ActionResult<Unit>> Update(Guid id, EditActivity.Command command)
        {
            command.Id = id;
            return await _mediator.Send(command);
        }
    }
```
11. **DELETE** Create a new class in your Application/Activities direcotry titled "DeleteActivity"
```cs
 public class DeleteActivity
    {
        //define the command param that will be needed to locate the individual activity
        public class Command : IRequest
        {
            public Guid Id { get; set; }
        }

        public class Handler : IRequestHandler<Command>
        {
            private readonly ApplicationDbContext _db;
            public Handler(ApplicationDbContext db)
            {
                _db = db;
            }
            public async Task<Unit> Handle(Command request, CancellationToken cancellationToken)
            {
                var activity = await _db.Activities.FindAsync(request.Id);

                if (activity == null)
                {
                    throw new Exception("Could not locate that activity");
                }

                _db.Remove(activity);

                var success = await _db.SaveChangesAsync() > 0;
                if (success)
                {
                    return Unit.Value;
                }
                throw new Exception("Problem saving changes");
            }
        }
    }
```
12. Controller
```cs
    [Route("api/[controller]")]
    [ApiController]
    public class ActivitiesController : ControllerBase
    {
        private readonly IMediator _mediator;
        public ActivitiesController(IMediator mediator)
        {
            _mediator = mediator;
        }

        [HttpDelete("{id}")]
        public async Task<ActionResult<Unit>> Delete(Guid id)
        {
            return await _mediator.Send(new DeleteActivity.Command{Id = id});
        }
    }
```
