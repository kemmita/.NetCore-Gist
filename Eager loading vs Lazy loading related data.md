1. First install Microsoft.EntityFrameworkCore.Proxies.
2. Below is the three classes we needed to alter in order to obtain eager loading.
```cs
    public class UserActivity
    {
        public string AppUserId { get; set; }
        //we needed to add virtual here
        public virtual AppUser AppUser { get; set; }
        public Guid ActivityId { get; set; }
        //and here
        public virtual Activity Activity { get; set; }
        public DateTime DateJoined { get; set; }
        public bool IsHost { get; set; }
    }
```
```cs
    public class AppUser : IdentityUser
    {
        public string DisplayName { get; set; }    
        //added virtual here
        public virtual ICollection<UserActivity> UserActivities { get; set; }
    }
```
```cs
    public class Activity
    {
        public Guid Id { get; set; }
        public string Title { get; set; }
        public string Description { get; set; }
        public string Category { get; set; }
        public string City { get; set; }
        public string Venue { get; set; }
        public DateTime Date { get; set; }
        [JsonProperty(PropertyName = "attendees")]
        //added virtual here
        public virtual ICollection<UserActivity> UserActivities { get; set; }
    }
```
3. Now we can change our entity query
```cs
//Eager ladoing
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

//Lazy
      public async Task<ActivityDto> Handle(Query request, CancellationToken cancellationToken)
            {
                var activity = await _db.Activities.FindAsync(request.Id);

                if (activity == null)
                {
                    throw new RestException(HttpStatusCode.NotFound, new { activity = "Activity Not Found" });
                }

                var toReturn = _mapper.Map<Activity, ActivityDto>(activity);
                return toReturn;
            }
```
