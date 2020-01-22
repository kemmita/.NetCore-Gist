1. Install nuget package AutoMapper.Extensions.Microsoft.DependencyInjection
2. In your statrtup class you will need to add a new serivce reference, referecning where the mapper profile will be located.
Now DotNet will know to look in the assembly containing the ActivitiesList.Handler class
```cs
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddMediatR(typeof(ActivitiesList.Handler).Assembly);
            services.AddAutoMapper(typeof(ActivitiesList.Handler).Assembly);
            services.AddMvc(options =>
            {
                var policy = new AuthorizationPolicyBuilder().RequireAuthenticatedUser().Build();
                options.Filters.Add(new AuthorizeFilter(policy));
            }).SetCompatibilityVersion(CompatibilityVersion.Version_2_2).AddFluentValidation(cfg =>
                {
                    cfg.RegisterValidatorsFromAssemblyContaining<CreateActivity>();
                });
            var builder = services.AddIdentityCore<AppUser>();

        }
```
3. Now lets create the mapper profile
```cs
  public class MappingProfile : Profile
    {
        public MappingProfile()
        {
            //As long as the props you want to map in the object have the same name and same data type, then this will suffice
            CreateMap<Activity, ActivityDto>();
            
            //If the two objects you want to map have props with the same datatype but diiferent names, then you will need to
            //explicilty configure them like below.
            CreateMap<UserActivity, AttendeeDto>()
            //in this case dest pertains to "AttendeeDto"
                .ForMember(dest => dest.Username, opt => opt.MapFrom(x => x.AppUser.UserName))
                .ForMember(dest => dest.DisplayName, opt => opt.MapFrom(x => x.AppUser.DisplayName))
                .ForMember(dest => dest.IsHost, opt => opt.MapFrom(x => x.IsHost));
        }
    }
```
4. Using the Mapper
```cs
       public async Task<ActivityDto> Handle(Query request, CancellationToken cancellationToken)
            {
                var activity = await _db.Activities.Include(ua => ua.UserActivities)
                    .ThenInclude(u => u.AppUser).FirstOrDefaultAsync(a => a.Id == request.Id);

                if (activity == null)
                {
                    throw new RestException(HttpStatusCode.NotFound, new { activity = "Activity Not Found" });
                }

                var toReturn = _mapper.Map<Activity, ActivityDto>(activity);
                return toReturn;
            }
```
5. Using the mapper with lists
```cs
    public async Task<List<ActivityDto>> Handle(Query request, CancellationToken cancellationToken)
            {
                var activities = await _db.Activities.Include(ua => ua.UserActivities)
                    .ThenInclude(u => u.AppUser).ToListAsync();

                return _mapper.Map<List<Activity>, List<ActivityDto>>(activities); ;
            }
```
