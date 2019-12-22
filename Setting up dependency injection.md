1. First create an interface 
```cs
public interface IJwtGenerator
    {
        string CreateToken(AppUser user);
    }
```
2. Nex create a concrete implementation
```cs
    public class JwtGenerator : IJwtGenerator
    {
        public string CreateToken(AppUser user)
        {
            var claims = new List<Claim>
            {
                new Claim(JwtRegisteredClaimNames.NameId, user.UserName)
            };

            var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("money"));
            var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha512Signature);

            var tokenDescriptor = new SecurityTokenDescriptor
            {
                Subject = new ClaimsIdentity(claims),
                Expires = DateTime.Now.AddDays(1000),
                SigningCredentials = creds
            };

            var tokenHandler = new JwtSecurityTokenHandler();

            var token = tokenHandler.CreateToken(tokenDescriptor);

            return tokenHandler.WriteToken(token);
        }
    }
```
3. Next go into the Startup class to register the interface and the class that implements the interface.
```cs
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddScoped<IJwtGenerator, JwtGenerator>();
        }
```
4. Then go to a class you wish to use it in
```cs
        public class test
        {
            private readonly IJwtGenerator _jwtGenerator;

            public Handler(IJwtGenerator jwtGenerator)
            {
                _jwtGenerator = jwtGenerator;
            }
            
            public async Task<User> Handle()
            {
                _jwtGenerator.Method();
            }
        }
```
