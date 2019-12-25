1. First download this nugget package
```
Microsoft.AspNetCore.Identity.EntityFrameworkCore
```
2. Create a new Model in your domain project, AppUser and extend IdentityUser. This will give us all of our default user props we will
need such as password, username etc. If we need to an additional prop, we can do so as we normally would. If we did not want to 
add any additional props, are model would look like below
```cs
using Microsoft.AspNetCore.Identity;

namespace Domain
{
    public class AppUser : IdentityUser
    {
        public string DisplayName { get; set; }      
    }
}
```
2.1 Create a UserObject in the Application Project so that we do not return all 10+ props returned when fetching the AppUser from the db.
```cs
   public class User
    {
        public string DisplayName { get; set; }
        public string Token { get; set; }
        public string Username { get; set; }
        public string Image { get; set; }
    }
```
3. In our persistence proeject, we will need to alter our ApplicationDbContext file a bit. Instead of inheriting from DbContext, we will
inherit from IdentityDbContext<AppUser> ensure to include the name of the user model, the one we craetd above.
```cs
public class ApplicationDbContext: IdentityDbContext<AppUser>
    {
        public ApplicationDbContext(DbContextOptions options): base(options)
        {

        }

        protected override void OnModelCreating(ModelBuilder builder)
        {
            //this is needed for everything to work!
            base.OnModelCreating(builder);
        }
    }
```
4. Then run your migration
```cs
add-migration AddIdentity -p Persistence -s API
update-database // may be uper case, I don't fucking know!
```
5. In our Startup class make the following addtions U may see some erros, and that is okay as we will be adding in the dependencies
later below
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
            services.AddMediatR(typeof(ActivitiesList.Handler).Assembly);
            services.AddMvc(options =>
            {
                var policy = new AuthorizationPolicyBuilder().RequireAuthenticatedUser().Build();
                options.Filters.Add(new AuthorizeFilter(policy));
            }).SetCompatibilityVersion(CompatibilityVersion.Version_2_2).AddFluentValidation(cfg =>
                {
                    cfg.RegisterValidatorsFromAssemblyContaining<CreateActivity>();
                });
            var builder = services.AddIdentityCore<AppUser>();
            var identityBuilder = new IdentityBuilder(builder.UserType, builder.Services);
            identityBuilder.AddEntityFrameworkStores<ApplicationDbContext>();
            identityBuilder.AddSignInManager<SignInManager<AppUser>>();

            var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("Super Secret Key"));
            services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme).AddJwtBearer(options =>
            {
                options.TokenValidationParameters = new TokenValidationParameters
                {
                    ValidateIssuerSigningKey = true,
                    IssuerSigningKey = key,
                    ValidateAudience = false,
                    ValidateIssuer = false
                };
            });
            services.AddScoped<IJwtGenerator, JwtGenerator>();
            services.AddScoped<IUserAccessor, UserAccessor>();
        }
        
        
        public void Configure(IApplicationBuilder app, IHostingEnvironment env)
        {
            app.UseMiddleware<ErrorHandlingMiddleware>();
            if (env.IsDevelopment())
            {
                //app.UseDeveloperExceptionPage();
            }
            else
            {
                // The default HSTS value is 30 days. You may want to change this for production scenarios, see https://aka.ms/aspnetcore-hsts.
                //app.UseHsts();
            }

            //app.UseHttpsRedirection();
            app.UseCors("CorsPolicy");
            app.UseAuthentication();
            app.UseMvc();
        }
```
6. We will now seed our database with test users. This will be done in the persistence project, file Seed.cs 
```cs
    public class Seed
    {
        public static async Task SeedData(ApplicationDbContext dbContext, UserManager<AppUser> userManager)
        {
            if (!userManager.Users.Any())
            {
                var users = new List<AppUser>
                {
                    new AppUser
                    {
                        DisplayName = "Bob",
                        UserName = "bob",
                        Email = "bob@test.com"
                    },
                    new AppUser
                    {
                        DisplayName = "Tom",
                        UserName = "tom",
                        Email = "tom@test.com"
                    },
                    new AppUser
                    {
                        DisplayName = "Jane",
                        UserName = "jane",
                        Email = "jane@test.com"
                    },
                };
                foreach (var user in users)
                {
                    await userManager.CreateAsync(user, "Password@1");
                }
            }
    }
```
7. Now in  our Programm class in the main method we will need to update the static method call to take in an instance of UserManager
and since the static method is async and the main method of the progrma is not, we will append the "Wait()" to the call
```cs
        public static void Main(string[] args)
        {
           
            var host = CreateWebHostBuilder(args).Build();
            using (var scope = host.Services.CreateScope())
            {
                var services = scope.ServiceProvider;
                try
                {
                    var context = services.GetRequiredService<ApplicationDbContext>();
                    var userManager = services.GetRequiredService<UserManager<AppUser>>();
                    context.Database.Migrate();
                    Seed.SeedData(context, userManager).Wait();
                }
                catch (Exception e)
                {
                    var logger = services.GetRequiredService<ILogger<Program>>();
                    logger.LogError(e, "An error has occured during Database Scaffolding!");
                }
            }
            host.Run();
        }
```
8. Add a new "project/class library", "Infrastructure" this lib will be responsible for handling Security related jobs.
9. Create a new directory securty and within place a new class titled JWT Generator. we will create the IJwtGenerator interfcae later
```cs
  public class JwtGenerator : IJwtGenerator
    {
        public string CreateToken(AppUser user)
        {
            var claims = new List<Claim>
            {
                new Claim(JwtRegisteredClaimNames.NameId, user.UserName)
            };

            var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("Super Secret Key"));
            var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha512Signature);

            var tokenDescriptor = new SecurityTokenDescriptor
            {
                Subject = new ClaimsIdentity(claims),
                Expires = DateTime.Now.AddDays(7),
                SigningCredentials = creds
            };

            var tokenHandler = new JwtSecurityTokenHandler();

            var token = tokenHandler.CreateToken(tokenDescriptor);

            return tokenHandler.WriteToken(token);
        }
    }
```
10. In the same security directory create a new class UserAccessor to fetch the current username concealed in the token after a user
logs in. IUserAccess is Created below
```cs
 public class UserAccessor : IUserAccessor
    {
        private readonly IHttpContextAccessor _httpContextAccessor;
        public UserAccessor(IHttpContextAccessor httpContextAccessor)
        {
            _httpContextAccessor = httpContextAccessor;
        }
        public string GetCurrentUsername()
        {
            var username = _httpContextAccessor.HttpContext.User?.Claims
                ?.FirstOrDefault(x => x.Type == ClaimTypes.NameIdentifier)?.Value;
            return username;
        }
    }
```
11. Now in the application lib, create a new direcotry for interfaces and create the IJwtGenerator interface. In the program class
above, we have already set up DI for this interface and its concrete implementation defined above "JwtGenerator"
```cs
using Domain;

namespace Application.Interfaces
{
    public interface IJwtGenerator
    {
        string CreateToken(AppUser user);
    }
}
```
12. Now create the IUserAccessor Interface. DI was done in the program class above already
```cs
namespace Application.Interfaces
{
    public interface IUserAccessor
    {
        string GetCurrentUsername();
    }
}
```
13. In the application lib in User directory we will create the Register Handler
```cs
    public class Register
    {
        public class Command : IRequest<User>
        {
            public string DisplayName { get; set; }
            public string Username { get; set; }
            public string Email { get; set; }
            public string Password { get; set; }
        }
        public class Handler : IRequestHandler<Command, User>
        {
            private readonly ApplicationDbContext _db;
            private readonly UserManager<AppUser> _userManager;
            private readonly IJwtGenerator _jwtGenerator;
            public Handler(ApplicationDbContext db, UserManager<AppUser> userManager, IJwtGenerator jwtGenerator)
            {
                _db = db;
                _userManager = userManager;
                _jwtGenerator = jwtGenerator;
            }

            public async Task<User> Handle(Command request, CancellationToken cancellationToken)
            {
                if (await _db.Users.AnyAsync(u => u.Email == request.Email))
                {
                    throw new RestException(HttpStatusCode.BadRequest, new {Email = "Email Already Exists"});
                }

                if (await _db.Users.AnyAsync(u => u.UserName == request.Username))
                {
                    throw new RestException(HttpStatusCode.BadRequest, new { Username = "Username Already Exists" });
                }

                var user = new AppUser
                {
                    DisplayName = request.DisplayName,
                    Email = request.Email,
                    UserName = request.Username
                };

                var results = await _userManager.CreateAsync(user, request.Password);

                if (results.Succeeded)
                {
                    return new User
                    {
                        DisplayName = user.DisplayName,
                        Token = _jwtGenerator.CreateToken(user),
                        Username = user.UserName,
                        Image = null
                    };
                }

                throw new Exception("Problem registering user.");
            }
        }
    }
```
14. Now Create the Registration Validator in the Validator directory in app lib
```cs
{
    public class RegistrationValidator : AbstractValidator<Register.Command>
    {
        public RegistrationValidator()
        {
            RuleFor(x => x.Username).NotEmpty().WithMessage("Username Required");
            RuleFor(x => x.DisplayName).NotEmpty().WithMessage("DisplayName Required");
            RuleFor(x => x.Email).NotEmpty().WithMessage("Email Required").EmailAddress().WithMessage("Valid Email Required");
            RuleFor(x => x.Password).NotEmpty().WithMessage("Password Required")
                .MinimumLength(6).WithMessage("Password Min Length Is 6 Characters")
                .Matches("[A-Z]").WithMessage("Password Must Contain An Upper Case Letter")
                .Matches("[a-z]").WithMessage("Password Must Contain A Lower Case Letter")
                .Matches("[0-9]").WithMessage("Password Must Contain A Number")
                .Matches("[^a-zA-Z0-9]").WithMessage("Password Must Contain A Special Character");
        }
    }
```
15. Now lets cerate the login handler
```cs
  public class Login
    {
        //specify what param is expected to be passed for this Query
        public class Query : IRequest<User>
        {
            public string Email { get; set; }
            public string Password { get; set; }
        }

        public class Handler : IRequestHandler<Query, User>
        {
            private readonly UserManager<AppUser> _userManager;
            private readonly SignInManager<AppUser> _signInManager;
            private readonly IJwtGenerator _jwtGenerator;

            public Handler(UserManager<AppUser> userManager, SignInManager<AppUser> signInManager, IJwtGenerator jwtGenerator)
            {
                _userManager = userManager;
                _signInManager = signInManager;
                _jwtGenerator = jwtGenerator;
            }
            //using Query defined above wih a param Id of type Guid, we will locate a single activity in the db associated with the Guid id.
            public async Task<User> Handle(Query request, CancellationToken cancellationToken)
            {
                var user = await _userManager.FindByEmailAsync(request.Email);

                if (user == null)
                {
                    throw new RestException(HttpStatusCode.Unauthorized);
                }

                var result = await _signInManager.CheckPasswordSignInAsync(user, request.Password, false);

                if (result.Succeeded)
                {
                    // Return Token
                    return new User
                    {
                        DisplayName = user.DisplayName,
                        Image = null,
                        Token = _jwtGenerator.CreateToken(user),
                        Username = user.UserName
                    };
                }
                else
                {
                    throw new RestException(HttpStatusCode.Unauthorized);
                }
            }
        }
    }
```
16. Create the Login validator
```cs
    public class LoginValidator : AbstractValidator<Login.Query>
    {
        public LoginValidator()
        {
            RuleFor(x => x.Email).NotEmpty().WithMessage("Email Required");
            RuleFor(x => x.Password).NotEmpty().WithMessage("Password Required");
        }
    }
```
17. Create the Current User handler to return the user who is currently signed in.
```cs
    public class CurrentUser
    {
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

                return new User
                {
                    DisplayName = user.DisplayName,
                    Username = user.UserName,
                    Token = _jwtGenerator.CreateToken(user),
                    Image = null
                };
            }
        }
    }
```
18. Lastly Create the User Controller
```cs
    [Route("api/[controller]")]
    [ApiController]
    public class UserController : BaseController
    {
        private readonly IMediator _mediator;
        public UserController(IMediator mediator)
        {
            _mediator = mediator;
        }

        [AllowAnonymous]
        [HttpPost("login")]
        public async Task<ActionResult<User>> Login(Login.Query query)
        {
           return await _mediator.Send(new Login.Query {Email = query.Email, Password = query.Password});
        }

        [AllowAnonymous]
        [HttpPost("Register")]
        public async Task<ActionResult<User>> Register(Register.Command command)
        {
            return await _mediator.Send(command);
        }

        [HttpGet]
        public async Task<ActionResult<User>> CurrentUser()
        {
            return await Mediator.Send(new CurrentUser.Query());
        }
    }
```
