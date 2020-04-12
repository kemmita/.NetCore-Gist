1. Let us first create our many to many self referencing relationship. Here is our join entity.
```cs
   public class UserFollowing
    {
        public string ObserverId { get; set; }
        // Specify virtual so that it becomes a navigation prop
        public virtual AppUser Observer { get; set; }
        public string TargetId { get; set; }
        // Specify virtual so that it becomes a navigation prop
        public virtual AppUser Target { get; set; }
    }
```
2. Below is the AppUser model
```cs
 public class AppUser : IdentityUser
    {
        public string DisplayName { get; set; }
        public string Bio { get; set; }
        public virtual ICollection<UserFollowing> Followings { get; set; }
        public virtual ICollection<UserFollowing> Followers { get; set; }
    }
```
3. Below is our dbcontext class
```cs
  public class ApplicationDbContext: IdentityDbContext<AppUser>
    {
        public ApplicationDbContext(DbContextOptions options): base(options)
        {

        }
        
        public DbSet<UserFollowing> Followings { get; set; }

        protected override void OnModelCreating(ModelBuilder builder)
        {
            base.OnModelCreating(builder);

            builder.Entity<UserFollowing>(b =>
            {
                b.HasKey(k => new { k.ObserverId, k.TargetId });

                b.HasOne(o => o.Observer)
                .WithMany(f => f.Followings)
                .HasForeignKey(o => o.ObserverId)
                .OnDelete(DeleteBehavior.Restrict);

                b.HasOne(o => o.Target)
                .WithMany(f => f.Followers)
                .HasForeignKey(o => o.TargetId)
                .OnDelete(DeleteBehavior.Restrict);
            });
        }
    }
```
4. Below is the handler for the logged in user to follow someone
```cs
    public class Add
    {
        public class Command : IRequest
        {
            public string Username { get; set; }
        }

        public class Handler : IRequestHandler<Command>
        {
            private readonly ApplicationDbContext _db;
            private readonly UserManager<AppUser> _userManager;
            private readonly IUserAccessor _userAccessor;
            public Handler(ApplicationDbContext db, UserManager<AppUser> userManager, IUserAccessor userAccessor)
            {
                _db = db;
                _userManager = userManager;
                _userAccessor = userAccessor;
            }
            public async Task<Unit> Handle(Command request, CancellationToken cancellationToken)
            {
                var observer = await _userManager.FindByNameAsync(_userAccessor.GetCurrentUsername());

                var target = await _userManager.FindByNameAsync(request.Username);

                if (target == null)
                        throw new RestException(HttpStatusCode.NotFound, new { target = "User Not Found" });

                var following = _db.Followings.Any(x => x.ObserverId == observer.Id && x.TargetId == target.Id);

                if (following)
                    throw new RestException(HttpStatusCode.BadRequest, new { following = "You already following this user" });


                await _db.Followings.AddAsync(new UserFollowing 
                {
                    Observer = observer,
                    Target = target,
                });

                var success = await _db.SaveChangesAsync() > 0;

                if (success)
                    return Unit.Value;

                throw new Exception("Problem creating following");
            }
        }
    }
```
5. Below is our handler for the current user to unfollow someone.
```cs
 public class Delete
    {
        public class Command : IRequest
        {
            public string Username { get; set; }
        }

        public class Handler : IRequestHandler<Command>
        {
            private readonly ApplicationDbContext _db;
            private readonly UserManager<AppUser> _userManager;
            private readonly IUserAccessor _userAccessor;
            public Handler(ApplicationDbContext db, UserManager<AppUser> userManager, IUserAccessor userAccessor)
            {
                _db = db;
                _userManager = userManager;
                _userAccessor = userAccessor;
            }
            public async Task<Unit> Handle(Command request, CancellationToken cancellationToken)
            {
                var observer = await _userManager.FindByNameAsync(_userAccessor.GetCurrentUsername());

                var target = await _userManager.FindByNameAsync(request.Username);

                if (target == null)
                    throw new RestException(HttpStatusCode.NotFound, new { target = "User Not Found" });

                var following = _db.Followings.Any(x => x.ObserverId == observer.Id && x.TargetId == target.Id);

                if (!following)
                    throw new RestException(HttpStatusCode.BadRequest, new { following = "You are not following this user" });


                _db.Followings.Remove(new UserFollowing
                {
                    Observer = observer,
                    Target = target,
                });

                var success = await _db.SaveChangesAsync() > 0;

                if (success)
                    return Unit.Value;

                throw new Exception("Problem removing following");
            }
        }
    }
```
6. Now we need to create our following controller
```cs
// Base Controller
   [Route("api/[controller]")]
    [ApiController]
    public class BaseController : ControllerBase
    {
        private IMediator _mediator;
        protected IMediator Mediator => _mediator ?? (_mediator = HttpContext.RequestServices.GetService<IMediator>());
    }
// Following controller inherits from the basecontroller
 public class FollowingController : BaseController
    {
        [HttpPost("{username}")]
        public async Task<ActionResult<Unit>> PostFollowing(string username, Add.Command command)
        {
            command.Username = username;
            return await Mediator.Send(command);
        }

        [HttpDelete("{username}")]
        public async Task<ActionResult<Unit>> DeleteFollowing(string username, Delete.Command command)
        {
            command.Username = username;
            return await Mediator.Send(command);
        }
    }
```

7. Quick and easy way to get follwoings and followers for the current user.
```cs
      public class Query : IRequest<User> { }

        public class Handler : IRequestHandler<Query, User>
        {
            private readonly UserManager<AppUser> _userManager;
            private readonly IUserAccessor _userAccessor;
            private readonly IJwtGenerator _jwtGenerator;

            public Handler(UserManager<AppUser> userManager, IUserAccessor userAccessor, IJwtGenerator jwtGenerator)
            {
                _userManager = userManager;
                _userAccessor = userAccessor;
                _jwtGenerator = jwtGenerator;
            }

            public async Task<User> Handle(Query request, CancellationToken cancellationToken)
            {
                var user = await _userManager.FindByNameAsync(_userAccessor.GetCurrentUsername());
                var following = user.Followings.Select(f => f.Target).ToList();
                var followers = user.Followings.Select(f => f.Observer).ToList();

                return new User
                {
                    DisplayName = user.DisplayName,
                    Username = user.UserName,
                    Token = _jwtGenerator.CreateToken(user),
                    Image = user.Photos.FirstOrDefault(p => p.IsMainPhoto)?.Url,
                    Followers = followers,
                    Following = following
                };
            }
        }
    }
```
8. Now lets return the followings and followers in a fancy way!
```cs
   public class Profile
    {
        public string DisplayName { get; set; }
        public string Username { get; set; }
        public string Image { get; set; }
        public string Bio { get; set; }
        // to check if a user is followed by the curent user.
        [JsonProperty(PropertyName = "following")]
        public bool IsFollowing { get; set; }
        public int FollowersCount { get; set; }
        public int FollowingCount { get; set; }
        public ICollection<Photo> Photos { get; set; }
    }
```
9. Create ProfileReader
```cs
   public interface IProfileReader
    {
        Task<Profile> ReadProfile(string username);
    }
    
        public class ProfileReader : IProfileReader
    {
        private readonly ApplicationDbContext _db;
        private readonly IUserAccessor _userAccessor; 
        public ProfileReader(ApplicationDbContext db, IUserAccessor userAccessor)
        {
            _db = db;
            _userAccessor = userAccessor;
        }
        public async Task<Profile> ReadProfile(string username)
        {
            var user = await _db.Users.SingleOrDefaultAsync(x => x.UserName == username);

            if (user == null)
                throw new RestException(HttpStatusCode.NotFound, new { user = "not found" });

            var currentUser = await _db.Users.SingleOrDefaultAsync(x => x.UserName == _userAccessor.GetCurrentUsername());

            var profile = new Profile
            {
                Photos = user.Photos,
                Bio = user.Bio ?? "Bio",
                DisplayName = user.DisplayName,
                Image = user.Photos.FirstOrDefault(p => p.IsMainPhoto)?.Url,
                Username = user.UserName,
                FollowersCount = user.Followers.Count(),
                FollowingCount = user.Followings.Count()
            };

            // chech to see if the current logged in user is following the user
            if (user.Followers.Any(x => x.ObserverId == currentUser.Id))
                profile.IsFollowing = true;

            return profile;
        }
    }
```
9. This handler can handle both fetchgin followers and followings depending on the predicate defined in the query string
```cs
    public class List
    {
        public class Query : IRequest<List<Profile>> 
        {
            public string Username { get; set; }
            public string Predicate { get; set; }
        }

        public class Handler : IRequestHandler<Query, List<Profile>>
        {

            private readonly ApplicationDbContext _db;
            private readonly IProfileReader _profileReader;
            public Handler(ApplicationDbContext db, IProfileReader profileReader)
            {
                _db = db;
                _profileReader = profileReader;
            }

            public async Task<List<Profile>> Handle(Query request, CancellationToken cancellationToken)
            {
                var userFollwings = new List<UserFollowing>();

                var profiles = new List<Profile>();

                switch (request.Predicate)
                {
                    case "followers":
                        userFollwings = await _db.Followings.Where(x => 
                            x.Target.UserName == request.Username).ToListAsync();
                        foreach (var follower in userFollwings)
                            profiles.Add(await _profileReader.ReadProfile(follower.Observer.UserName));
                        break;
                    case "following":
                        userFollwings = await _db.Followings.Where(x =>
                            x.Observer.UserName == request.Username).ToListAsync();
                        foreach (var follower in userFollwings)
                            profiles.Add(await _profileReader.ReadProfile(follower.Target.UserName));
                        break;
                    default:
                        break;
                }

                return profiles;
            }
        }
    }
```
10. Now the controller method. Specify followers or follwing in the query like ?predicate=followers
```cs
        [HttpGet("{username}")]
        public async Task<ActionResult<List<Profile>>> GetFollowingData(string username, string predicate) 
        {
            return await Mediator.Send(new List.Query {Predicate = predicate, Username = username });
        }
```
