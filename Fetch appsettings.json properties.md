1. Create new object in appsettings.json
```json
  "SmtpOptions": {
    "Host": "imail.com",
    "Username": "username",
    "Password": "password",
    "Port": 2000
  }
```
2. Next create a model that models the object above.
```cs
  public class SmtpOptions
    {
        public string Host { get; set; }
        public string Username { get; set; }
        public string Password { get; set; }
        public int Port { get; set; }
    }
```
3. Next register in Startup.cs
```cs
services.Configure<SmtpOptions>(Configuration.GetSection("SmtpOptions"));
```
4. Now we can inject these properties into a class usiing IOptions
```cs
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
