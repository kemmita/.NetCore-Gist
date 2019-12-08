1. First we will need to go ahead and install a nuget package in our Application project.
```
FluentValidation.AspNetCore
```
2. In the Startup.cs file we will need to alter our config service metod
```cs
        public void ConfigureServices(IServiceCollection services)
        {
            //we want to target our assembly of Application, so pick any public class within that assembly, and
            //fluent validation will be configured for that assembly
            services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_2).AddFluentValidation(cfg =>
                {
                    cfg.RegisterValidatorsFromAssemblyContaining<CreateActivity>();
                });
        }
```
3. We want to validate a command to create an activity, below is the controller
```cs
       [HttpPost("")]
        public async Task<ActionResult<Unit>> Create(CreateActivity.Command command)
        {
            return await _mediator.Send(command);
        }
```
4. Below is the CreateActivity class. We want to check the Command before it hits our Handler
```cs
    public class CreateActivity
    {
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
5. Within the same assmebly, "application lib" create a new directory, Validators, we will call the class, CreateCommand Validator.
```cs
   public class CreateCommandValidator : AbstractValidator<CreateActivity.Command>
    {
        public CreateCommandValidator()
        {
            RuleFor(x => x.City).NotEmpty().WithMessage("City must be provided");
            RuleFor(x => x.Title).NotEmpty().WithMessage("Title must be provided");
            RuleFor(x => x.Description).NotEmpty().WithMessage("Description must be provided");
            RuleFor(x => x.Category).NotEmpty().WithMessage("Category must be provided");
            RuleFor(x => x.Venue).NotEmpty().WithMessage("Venue must be provided");
            RuleFor(x => x.Date).NotEmpty().WithMessage("Date must be provided");
        }
    }
```
6. Thats it!
