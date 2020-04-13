1. Below is our mapping profile, we want to check and see if we are following a user and return a boolean of true if so.
```
    // DTO we are mapping our domain object to. Our domain object will not have a prop of following
    public class AttendeeDto
    {
        public string Username { get; set; }
        public string DisplayName { get; set; }
        public string Image { get; set; }
        public bool IsHost { get; set; }
        // we need to supply true or false for this prop
        public bool Following { get; set; }
    }
    
      public class MappingProfile : Profile
    {
        public MappingProfile()
        {
            CreateMap<UserActivity, AttendeeDto>()
                .ForMember(dest => dest.Username, opt => opt.MapFrom(x => x.AppUser.UserName))
                .ForMember(dest => dest.DisplayName, opt => opt.MapFrom(x => x.AppUser.DisplayName))
                // here we call our resolver
                .ForMember(dest => dest.Following, opt => opt.MapFrom<FollowingResolver>());
        }
    }
```
2. Below is the custom resolver
```cs
 public class FollowingResolver : IValueResolver<UserActivity, AttendeeDto, bool>
    {
        private readonly ApplicationDbContext _db;
        private readonly IUserAccessor _userAccessor;
        public FollowingResolver(ApplicationDbContext db, IUserAccessor userAccessor)
        {
            _db = db;
            _userAccessor = userAccessor;
        }
        public bool Resolve(UserActivity source, AttendeeDto destination, bool destMember, ResolutionContext context)
        {
            var currentUser = _db.Users.FirstOrDefaultAsync(x => x.UserName == _userAccessor.GetCurrentUsername()).Result;

            if (currentUser.Followings.Any(x => x.TargetId == source.AppUserId))
                return true;

            return false;
        }
    }
```
