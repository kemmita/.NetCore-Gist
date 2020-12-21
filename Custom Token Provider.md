1. Let's say we want to generate and validate tokens, a token that would be sent to a users email that we would have them send back to us to validate. First create a token provider call that will ineherit from DataProtectorTokenProvider.
```cs
  public class SubmissionTokenProvider<TUser>: DataProtectorTokenProvider<TUser> where TUser : class
    {
        // Note the IOptions will contain the SubmissionTokenProviderOptions class that we will create below.
        public FilmTokenProvider(IDataProtectionProvider dataProtectionProvider, IOptions<SubmissionTokenProviderOptions> options, ILogger<DataProtectorTokenProvider<TUser>>                    logger) : base(dataProtectionProvider, options, logger)
        {
        }
    }
```
2. SubmissionTokenProviderOptions class
```cs
  public class SubmissionTokenProviderOptions : DataProtectionTokenProviderOptions
    {
    }
```
3. Now in your Startup.cs ConfigureServices method, we will need to register the token provider and supply it with a unuiqe name
```cs
var builder = services.AddIdentityCore<AppUser>().AddDefaultTokenProviders().AddTokenProvider<SubmissionTokenProvider<AppUser>>("Submission");
```
4. Now to use them
```cs
   public async Task<string> GenerateSubmissionTokenAsync()
        {
            var user = await _db.AppUsers.FirstOrDefaultAsync(x => x.UserName == _userAccessor.GetCurrentUsername());
            return await _userManager.GenerateUserTokenAsync(user, "Submission", "Submission");
        }

        public async Task<bool> VerifySubmissionTokenAsync(string token)
        {
            var user = await _db.AppUsers.FirstOrDefaultAsync(x => x.UserName == _userAccessor.GetCurrentUsername());
            return await _userManager.VerifyUserTokenAsync(user, "Submission", "Submission", token);
        }
```
