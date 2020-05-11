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
            // 3.1 change AddMVC to AddControllers, the rest is the same
            services.AddMvc(options =>
            {
                var policy = new AuthorizationPolicyBuilder().RequireAuthenticatedUser().Build();
                options.Filters.Add(new AuthorizeFilter(policy));
                //in 3.1 may need to remove SetCompatibilityVersion before AddFluentValidation
            }).SetCompatibilityVersion(CompatibilityVersion.Version_2_2).AddFluentValidation(cfg =>
                {
                    cfg.RegisterValidatorsFromAssemblyContaining<CreateActivity>();
                });
            var builder = services.AddIdentityCore<AppUser>();
            var identityBuilder = new IdentityBuilder(builder.UserType, builder.Services);
            identityBuilder.AddEntityFrameworkStores<ApplicationDbContext>();
            identityBuilder.AddSignInManager<SignInManager<AppUser>>();

            var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("Super Secret Key"));
            // will need to install JwtBearerDefaults nugget to this api proj for 3.1
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
        
        //2.2
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
        
        //3.1
                public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            app.UseMiddleware<ErrorHandlingMiddleware>();
            
            //using microsoft.extensions.hsoting
            if (env.IsDevelopment())
            {
                //app.UseDeveloperExceptionPage();
            }
            else
            {
                // The default HSTS value is 30 days. You may want to change this for production scenarios, see https://aka.ms/aspnetcore-hsts.
                //app.UseHsts();
            }
            app.UseRouting
            app.UseCors("CorsPolicy");
            app.UseAuthentication();
            app.UseAuthorization(); 
            
            app.useendpoints
            
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
19. Adding Facbook login, frist go to "https://developers.facebook.com/" and register a new app. Make sure to grab the app id and secret
```
App ID
############
App Secret
##########################
```
20. Next we will go to the react project and add a new react component
```
npm i react-facebook-login @types/react-facebook-login
```
21. First we will make a change to our loginform component.
```ts
import React, {useContext} from 'react';
import {observer} from "mobx-react-lite";
import {RootStoreContext} from "../../app/stores/rootStore";
import {Form as FinalForm, Field} from 'react-final-form';
import {Button, Divider, Form, Header} from 'semantic-ui-react';
import TextInput from "../../app/common/form/TextInput";
import {IUserFormValues} from "../../app/Interfaces/user";
import {FORM_ERROR} from "final-form";
import {combineValidators, isRequired} from "revalidate";
import ErrorMessage from "../../app/common/form/ErrorMessage";
import SocialLogin from "./SocialLogin";

const validate = combineValidators({
   email: isRequired('email'),
   password: isRequired('password')
});

const LoginForm: React.FC = (props) => {
    const rootStore = useContext(RootStoreContext);
    const {userStore} = rootStore;
    return (
        <FinalForm
            validate={validate}
            onSubmit={(values:IUserFormValues) => userStore.login(values).catch(err => ({
                [FORM_ERROR] : err
            }))}
            render={({handleSubmit, submitting, form, submitError, invalid, pristine, dirtySinceLastSubmit}) =>(
                <Form onSubmit={handleSubmit} error>
                    <Header content='Login Now!' textAlign='center' style={{color: '#1E6F9D'}} size={'large'}/>
                    <Field name='email' component={TextInput} placeholder='Email' />
                    <Field name='password' type='password' component={TextInput} placeholder='Password' />
                    {submitError && !dirtySinceLastSubmit && <ErrorMessage error={submitError.message} text={'Invalid email or password'}/>}
                    <br/>
                    <Button fluid disabled={invalid && !dirtySinceLastSubmit || pristine} style={{backgroundColor: '#1E6F9D', color: 'white'}} content='Login' loading={submitting}/>
                    // below we add our social login component
                    <Divider horizontal>Or</Divider>
                    <SocialLogin fbCallback={userStore.fbLogin}/>
                </Form>
            )}
        />
    );
};

export default observer(LoginForm);

```
22. Below we will create our social login component.
```ts
import React from 'react';
import {observer} from "mobx-react-lite";
import FacebookLogin from 'react-facebook-login/dist/facebook-login-render-props'
import {Button, Icon} from "semantic-ui-react";

interface IProps {
    fbCallback: (response: any) => void;
}

const ActivityDetails: React.FC<IProps> = (props) => {
    return (
        <div>
            <FacebookLogin
                appId='534231783929420'
                fields='name,email,picture'
                callback={props.fbCallback}
                render={(renderProps: any) =>(
                    <Button onClick={renderProps.onClick} type='button' fluid color={'facebook'}>
                        <Icon name={'facebook'} />
                        Login Mit Facebook
                    </Button>
                )}
            />
        </div>
    );
};

export default observer(ActivityDetails);
```
23. User Store Action
```ts
    @action fbLogin = async (response: any) =>{
        try {
            const user = await agent.User.fbLogin(response.accessToken);
            runInAction('Return user after successful login', () =>{
                this.user = user;
                this.rootStore.commonStore.setToken(user.token);
                this.rootStore.modalStore.closeModal();
                history.push('/activities');
            });
        } catch (e) {
            console.log(e)
        }
    };
```
24. agent.ts
```ts
import axios, { AxiosResponse } from 'axios';
import { history } from '../..';
import { toast } from 'react-toastify';
import {IActivitiesEnvelope, IActivity} from "../Interfaces/activity";
import {IUser, IUserFormValues} from "../Interfaces/user";
import {IPhoto, IProfile} from "../Interfaces/profile";

axios.defaults.baseURL = 'https://localhost:44396/api';

axios.interceptors.request.use((config) =>{
    const token = window.localStorage.getItem('jwt');
    if (token){
        config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
}, error => {return Promise.reject(error)});

axios.interceptors.response.use(undefined, error => {
    if (error.message === 'Network Error' && !error.response) {
        toast.error('Network error - make sure API is running!')
    }
    const {status, data, config} = error.response;
    if (status === 400 || 404 && config.method === 'get' && data.errors.hasOwnProperty('id')) {
        history.push('/notfound')
    }
    if (status === 500) {
        toast.error('Server error - check the terminal for more info!')
    }
    if (status === 401) {
        toast.error('Pleas login again to continue browsing!')
    }
    throw error;
});

const responseBody = (response: AxiosResponse) => response.data;

const requests = {
    post: (url: string, body: {}) => axios.post(url, body).then(responseBody),
};

const User = {
  login: (user: IUserFormValues): Promise<IUser> => requests.post('/user/login', user),
  register: (user: IUserFormValues): Promise<IUser> => requests.post('/user/register', user),
  fbLogin: (accessToken: string) => requests.post(`/user/External/login`, {accessToken}),
};

export default {
    Activities,
    User
}
```
25. API Controller action
```cs
 [Route("api/[controller]")]
    [ApiController]
    public class UserController : BaseController
    {
        private readonly IMediator _mediator;
        public UserController(IMediator mediator, IPhotoAccessor photoAccessor)
        {
            _mediator = mediator;
        }

        [AllowAnonymous]
        [HttpPost("External/login")]
        public async Task<ActionResult<User>> ExternalLogin(ExternalLogin.Query query)
        {
            return await _mediator.Send(query);
        }
    }
```
26. Handler class
```cs
public class ExternalLogin
    {
        //specify what param is expected to be passed for this Query
        public class Query : IRequest<User>
        {
            public string AccessToken { get; set; }
        }

        public class Handler : IRequestHandler<Query, User>
        {
            private readonly UserManager<AppUser> _userManager;
            private readonly IJwtGenerator _jwtGenerator;
            private readonly IFacebookAccessor _facebookAccessor;

            public Handler(UserManager<AppUser> userManager, IJwtGenerator jwtGenerator, IFacebookAccessor facebookAccessor)
            {
                _userManager = userManager;
                _jwtGenerator = jwtGenerator;
                _facebookAccessor = facebookAccessor;
            }
            //using Query defined above wih a param Id of type Guid, we will locate a single activity in the db associated with the Guid id.
            public async Task<User> Handle(Query request, CancellationToken cancellationToken)
            {
                var userInfo = await _facebookAccessor.FacebookLogin(request.AccessToken);

                if (userInfo == null)
                    throw new RestException(HttpStatusCode.BadRequest, new { User = "problem validating facebook token." });

                var user = await _userManager.FindByIdAsync(userInfo.Id);

                if (user == null)
                {
                    user = new AppUser
                    {
                        DisplayName = userInfo.Name,
                        Id = userInfo.Id,
                        Email = userInfo.Email,
                        UserName = "fb_" + userInfo.Id
                    };
                    var photo = new Photo
                    {
                        Id = "fb_" + userInfo.Id,
                        IsMainPhoto = true,
                        Url = userInfo.Picture.Data.Url
                    };

                    user.Photos.Add(photo);

                    var result = await _userManager.CreateAsync(user);

                    if (!result.Succeeded)
                        throw new RestException(HttpStatusCode.BadRequest, new { User = "Problem creating new user via facebook" });
                }

                return new User
                {
                    DisplayName = user.DisplayName,
                    Image = user.Photos.FirstOrDefault(x => x.IsMainPhoto == true)?.Url,
                    Token = _jwtGenerator.CreateToken(user),
                    Username = user.UserName
                };
            }
        }
    }
```
27. Below is our FacebookUserInfo object we needed to create in order to parse the data returned correctly.
```cs
   public class FacebookUserInfo
    {
        public string Email { get; set; }
        public string Id { get; set; }
        public string Name { get; set; }
        public FacebookPictureData Picture { get; set; }
    }

    public class FacebookPictureData
    {
        public FacebookPicture Data { get; set; }
    }

    public class FacebookPicture
    {
        public string Url { get; set; }
    }
```
28. We wil now create our interface and implementer class for said interfcae, ensure to add these via startup class
```cs
   public interface IFacebookAccessor
    {
        Task<FacebookUserInfo> FacebookLogin(string accessToken);
    }
```
29. FacebookAccessor class
```cs
    public class FacebookAccessor : IFacebookAccessor
    {
        private readonly HttpClient _httpClient;

        private readonly IOptions<FacebookAppSettings> _config;
        public FacebookAccessor(IOptions<FacebookAppSettings> config)
        {
            _config = config;

            _httpClient = new HttpClient
            {
                BaseAddress = new System.Uri("https://graph.facebook.com/")
            };

            _httpClient.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
        }

        public async Task<FacebookUserInfo> FacebookLogin(string accessToken)
        {
            // verify token is valid using facebook
            var verifiedToen = await _httpClient.GetAsync($"debug_token?input_token={accessToken}&access_token={_config.Value.AppId}|{_config.Value.AppSecret}");

            if (!verifiedToen.IsSuccessStatusCode)
                return null;

            return await GetAsync<FacebookUserInfo>(accessToken, "me", "fields=name,email,picture.width(100).height(100)");
        }

        private async Task<T> GetAsync<T>(string accessToken, string endpoint, string args)
        {
            // Go and get users infor from facebook 
            var response = await _httpClient.GetAsync($"{endpoint}?access_token={accessToken}&{args}");

            if (!response.IsSuccessStatusCode)
                return default(T);

            var result = await response.Content.ReadAsStringAsync();

            return JsonConvert.DeserializeObject<T>(result);
        }
    }
```
30. Now lets add email confirmation for our registration, you will need to create an account with MailJet before doing this.
after you setup an account add the nuget package to the respected libs and api.
```
Mailjet.Api
```
31. Add a new section to our appsetings.json file
```json
  "Smtp": {
    "Host": "in-v3.mailjet.com",
    "Username": "dabfbb37232876666de959a659de58cc",
    "Password": "d61aa4bd876fe7d53b756e4638975477",
    "Port": 587
  },
```
32. Create a new object to match the props above, we are going to strongly type this.
```cs
    public class SmtpOptions
    {
        public string Host { get; set; }
        public string Username { get; set; }
        public string Password { get; set; }
        public int Port { get; set; }
    }
```
33. Lets now create our emnail sender class
```cs
  public interface IEmailSender
    {
        Task SendEmailAsync(string from, string to, string subject, string message);
    }
    
      public class EmailSender : IEmailSender
    {
        private readonly IOptions<SmtpOptions> _config;

        public EmailSender(IOptions<SmtpOptions> config)
        {
            _config = config;
        }

        public async Task SendEmailAsync(string from, string to, string subject, string message)
        {
            var mailMessage = new MailMessage(from, to, subject, message);

            using (var client = new SmtpClient(_config.Value.Host, _config.Value.Port)
            {
                Credentials = new NetworkCredential(_config.Value.Username, _config.Value.Password)
            })
            {
                await client.SendMailAsync(mailMessage);
            }
        }
    }
  
```
34. Below I notate how I modified the register class
```cs
       public class Handler : IRequestHandler<Command, User>
        {
            private readonly ApplicationDbContext _db;
            private readonly UserManager<AppUser> _userManager;
            private readonly IJwtGenerator _jwtGenerator;
            // add the email sender
            private readonly IEmailSender _emailSender;
            public Handler(ApplicationDbContext db, UserManager<AppUser> userManager, IJwtGenerator jwtGenerator, IEmailSender emailSender)
            {
                _db = db;
                _userManager = userManager;
                _jwtGenerator = jwtGenerator;
                _emailSender = emailSender;
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
                    // fetch the user we just created
                    user = await _userManager.FindByNameAsync(user.UserName);
                    // create a email token
                    var token = await _userManager.GenerateEmailConfirmationTokenAsync(user);
                    // create the link for the email
                    var confirmatrionLink = new Uri($"https://localhost:44396/api/user/confirmEmail?userId={user.Id}&token={HttpUtility.UrlEncode(token)}");
                    // send the email
                    await _emailSender.SendEmailAsync("russellkemmitdeveloper@gmail.com", user.Email, "Kemmittech Email Confirmation", $"Please confirm your account, by selecting this link: {confirmatrionLink}");

                    return new User
                    {
                        DisplayName = user.DisplayName,
                        Token = _jwtGenerator.CreateToken(user),
                        Username = user.UserName,
                        Image = user.Photos.FirstOrDefault(p => p.IsMainPhoto)?.Url
                     };                                                                                                                                                                                                                                                       
                }

                throw new Exception("Problem registering user.");
            }
        }
```
35. Below I notate how I modified the login class
```cs
            public async Task<User> Handle(Query request, CancellationToken cancellationToken)
            {
                var user = await _userManager.FindByEmailAsync(request.Email);

                if (user == null)
                {
                    throw new RestException(HttpStatusCode.Unauthorized);
                }

                var result = await _signInManager.CheckPasswordSignInAsync(user, request.Password, false);
                // check if the email is confirmed
                var emailConfirmed = await _userManager.IsEmailConfirmedAsync(user);
                
                // if the email is not confirmed throw an exception
                if (!emailConfirmed)
                    throw new RestException(HttpStatusCode.Unauthorized, new { Email = "You must confirm your email" });

                if (result.Succeeded)
                {
                    // Return Token
                    return new User
                    {
                        DisplayName = user.DisplayName,
                        Image = user.Photos.FirstOrDefault(x => x.IsMainPhoto == true)?.Url,
                        Token = _jwtGenerator.CreateToken(user),
                        Username = user.UserName
                    };
                }
                else
                {
                    throw new RestException(HttpStatusCode.Unauthorized);
                }
            }
```
36. In the user controller, we will create a new action that will act as the endpoint hit when the user selects the 
confirm link in the email. You must include the [FromQuery] in order to get the props.
```cs
        [AllowAnonymous]
        [HttpGet("confirmEmail")]
        public async Task<ActionResult<Unit>> ConfirmEmail([FromQuery]ConfirmEmail.Command command)
        {
             // go and confirm the email   
             await _mediator.Send(command);
            // redirect the user to an address of your choosing.
            return Redirect("http://localhost:3000");
        }
```
37.Below is the confirm email address
```cs
    public class ConfirmEmail
    {
        public class Command : IRequest
        {
            public string Token { get; set; }
            public string UserId { get; set; }
        }
        public class Handler : IRequestHandler<Command>
        {
            private readonly ApplicationDbContext _db;
            private readonly UserManager<AppUser> _userManager;
            public Handler(ApplicationDbContext db, UserManager<AppUser> userManager)
            {
                _db = db;
                _userManager = userManager;
            }

            public async Task<Unit> Handle(Command request, CancellationToken cancellationToken)
            {
                var user = await _userManager.FindByIdAsync(request.UserId);

                if (user == null)
                    throw new RestException(HttpStatusCode.NotFound, new { User = "User Not Found" });
                
                var result = await _userManager.ConfirmEmailAsync(user, request.Token);

                if (result.Succeeded)
                    return Unit.Value;

                throw new Exception("Problem registering user.");
            }
        }
}
```
38. Below is the changes I made to the startup class
```cs
            // needed to append AddDefaultTokenProviders()
            var builder = services.AddIdentityCore<AppUser>().AddDefaultTokenProviders();
            // added this section to require the email confirmation "1"
            services.Configure<IdentityOptions>(options =>
            {
                options.SignIn.RequireConfirmedEmail = true;
            });
            // added our new email sender class and strongly typed smtp options 
            services.AddScoped<IEmailSender, EmailSender>();
            services.Configure<SmtpOptions>(Configuration.GetSection("Smtp"));
```
39. If you want to see the edits made for the front end visit https://github.com/kemmita/React-Gist/blob/master/Consuming%20ASP%20Identity.md 
