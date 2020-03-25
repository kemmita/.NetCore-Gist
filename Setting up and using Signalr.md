1. First we need to head to our startup class and add signalr as a service. We also need to add the current username to the hub context
in the ConfigureServices method.
```cs
          public void ConfigureServices(IServiceCollection services)
        {
            services.AddDbContext<ApplicationDbContext>(opt =>
            {
                opt.UseLazyLoadingProxies();
                opt.UseSqlServer(Configuration.GetConnectionString("DefaultConnection"));

            });
            services.AddCors(opt =>
            {
                opt.AddPolicy("CorsPolicy", policy =>
                    {
          //We needed to append the AllowCredentials, for the token to be sent correctly to the hub.                    
                        policy.AllowAnyHeader().AllowAnyMethod().WithOrigins("http://localhost:3000").AllowCredentials();
                    });
            });
            services.AddMediatR(typeof(ActivitiesList.Handler).Assembly);
            services.AddAutoMapper(typeof(ActivitiesList.Handler).Assembly);
            services.AddSignalR();
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
            services.AddAuthorization(opt =>
            {
                opt.AddPolicy("IsActivityHost", policy =>
                {
                    policy.Requirements.Add(new IsHostRequirement());
                });
            });
            services.AddTransient<IAuthorizationHandler, IsHostRequirementHandler>();
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
                // Here we prvoide the value of accessTOken to our "HUB" context.token, this will contain the current username.
                options.Events = new JwtBearerEvents
                {
                    OnMessageReceived = context =>
                    {
                        var accessToken = context.Request.Query["access_token"];
                        var path = context.HttpContext.Request.Path;
                        if (!string.IsNullOrEmpty(accessToken) && (path.StartsWithSegments("/chat")))
                        {
                            context.Token = accessToken;
                        }
                        return Task.CompletedTask;
                    }
                };
            });
            services.AddScoped<IJwtGenerator, JwtGenerator>();
            services.AddScoped<IUserAccessor, UserAccessor>();
            services.AddScoped<IPhotoAccessor, PhotoAccessor>();
            services.Configure<CloudinarySettings>(Configuration.GetSection("Cloudinary"));
        }
```
2. In the same file, lets update our ConfigureMethod
```cs
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
            app.UseSignalR(route =>
            {
                //ChatHub is the name of the controller/hub!
                route.MapHub<ChatHub>("/chat");
            });
        }
```
3. Below is ouir chathub
```cs
 public class ChatHub : Hub
    {
        private readonly IMediator _mediator;
        public ChatHub(IMediator mediator)
        {
            _mediator = mediator;
        }

        //this will act just like our controller endpoint. 
        public async Task SendComment(Create.Command command)
        {
            var username = CurrentUserName();

            command.Username = username;
             
             //send to our mediator handler per-normal
            var comment = await _mediator.Send(command);
            
            //All of our activities belong to the same hub, so we need to use Groups. The Groups
            //will be defined by the activityId
            await Clients.Group(command.ActivityId.ToString()).SendAsync("ReceiveComment", comment);
        }

        //When the connection to the hub is established "UseEffect for activity N", this method will be ran. 
        //The activityId will be passed in as the groupName. The Individual Context.COnnectionId that each user 
        //will create, will then be added to the group.
        public async Task AddToGroup(string groupName)
        {
            await Groups.AddToGroupAsync(Context.ConnectionId, groupName);

            await Clients.Group(groupName).SendAsync("Send", $"{CurrentUserName()} has joined the group.");
        }
          
        //When a user leaves the room, their connectionId will be removed from that ActivityId/Group  
        public async Task RemoveFromGroup(string groupName)
        {
            await Groups.RemoveFromGroupAsync(Context.ConnectionId, groupName);

            await Clients.Group(groupName).SendAsync("Send", $"{CurrentUserName()} has left the group.");
        }

        private string CurrentUserName()
        {
            return Context.User?.Claims?.FirstOrDefault(x => x.Type == ClaimTypes.NameIdentifier)?.Value;
        }
    }
```
4. Now lets head over to react and we will first need to instal needed packages.
```
npm i @microsoft/signalr
```
5. Now we will go to our activity store, but instead of calling an API endpoint, we will invoke a method on our server using signalr
```ts
    //when the activitiesDetail page is loaded, it will trigger a HubConnection
    @observable.ref hubConnection: HubConnection | null = null;

    @action createHubConnection = (activityId: string) =>{
        this.hubConnection = new HubConnectionBuilder()
            .withUrl('https://localhost:44396/chat', {
                accessTokenFactory: () => this.rootStore.commonStore.token!
            })
            .configureLogging(LogLevel.Information)
            .build();

        this.hubConnection.start()
            .then(() => console.log(this.hubConnection!.state))
            .then(() =>{
                this.hubConnection!.invoke('AddToGroup', activityId)
            })
            .catch(e => console.log('Error establishing signal r connection', e));

        this.hubConnection.on('ReceiveComment', comment => {
            runInAction(() =>{
                this.activity!.comments.push(comment);
            });
        });

        this.hubConnection.on('Send', message =>{
            toast.info(message);
        });
    };

    @action stopHubConnection = () =>{
        this.hubConnection!.invoke('RemoveFromGroup', this.activity!.id)
            .then(() =>{
                this.hubConnection!.stop();
            })
            .then(() => console.log('Connection Stopped'))
            .catch(() => console.log('Error stopping hub connection'));
    };

    @action addComment = async (values: any) =>{
        values.activityId = this.activity!.id;
        try {
            await this.hubConnection!.invoke('SendComment', values)
        } catch (e) {
            console.log(e);
        }
    };
```
6. Here in our activity chat component
```ts
interface IProps {
    activityId: string
}

const ActivityDetailedChat: React.FC<IProps> = (props) => {
    const rootStore = useContext(RootStoreContext);
    const {activityStore} = rootStore;

    //useEffect will go off and create the hub connection.
    useEffect(() =>{
        activityStore.createHubConnection(props.activityId);
        return() =>{
            activityStore.stopHubConnection();
        };
    }, [activityStore]);


    return (
        <Fragment>
            <Segment
                textAlign='center'
                attached='top'
                inverted
                color='teal'
                style={{ border: 'none' }}
            >
                <Header>Chat about this event</Header>
            </Segment>
            <Segment attached>
                <Comment.Group>
                    {activityStore.activity && activityStore.activity.comments && activityStore.activity!.comments.map((comment) =>(
                        <Comment key={comment.id}>
                            <Comment.Avatar src={comment.image || '/assets/user.png'} />
                            <Comment.Content>
                                <Comment.Author as='a'>{comment.displayName}</Comment.Author>
                                <Comment.Metadata>
                                    <div>{formatDistance(comment.createdAt, new Date())}</div>
                                </Comment.Metadata>
                                <Comment.Text>{comment.body}</Comment.Text>
                            </Comment.Content>
                        </Comment>
                    ))}
                        <FinalForm
                            onSubmit={activityStore.addComment}
                            render={(props) =>(
                                <Form reply onSubmit={() => props.handleSubmit()!.then(() => props.form.reset())}>
                                    {activityStore.activity!.comments.length > 0 ?
                                        <Field name={'body'} component={TextAreaInput} rows={3}
                                               placeholder={'Write Something Beautiful!'}/>
                                        :
                                        <Field name={'body'} component={TextAreaInput} rows={3}
                                               placeholder={'Be the first to write something cool!'}/>
                                    }
                                    <Button
                                        content='Add Reply'
                                        labelPosition='left'
                                        icon='edit'
                                        primary
                                        type={'submit'}
                                        disabled={props.invalid || props.pristine}
                                        loading={props.submitting}
                                    />
                                </Form>
                            )}/>
                </Comment.Group>
            </Segment>
        </Fragment>
    );
};
```
