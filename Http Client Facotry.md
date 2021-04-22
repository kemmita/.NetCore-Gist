1. Install NuGet package
```
System.Net.Http.Json
```
1.5. appsettings.json
```json
 "AllowedHosts": "*",
  "test_api": "https:testapi.com"
```
2. Startup.cs
```cs
       public void ConfigureServices(IServiceCollection services)
        {
            services.AddControllersWithViews();
            services.AddRazorPages();
            // add http clinet
            services.AddHttpClient();
            // add http clinet with specified endpoing from appsettings.json
            services.AddHttpClient("test", config =>
            {
                config.BaseAddress = new Uri(Configuration.GetValue<string>("test_api"));
            });
            // add httpclientfactoryservice and service that will utilize the httpclientfactoryservice
            services.AddScoped<IHttpClientFactoryService, HttpClientFactoryService>();
            services.AddScoped<ICommonService, CommonService>();
        }
```
3. Create interface for your http clinet factory service class
```cs
 public interface IHttpClientFactoryService
    {
        Task<TResponse> PostAsync<TResponse>(IFormFile image, string url, string client);
    }
```
4. Class that implements interface above
```cs
 public class HttpClientFactoryService : IHttpClientFactoryService
    {
        private readonly IHttpClientFactory _httpClient;
        private JsonSerializerOptions DefaultJsonSerializerOptions => 
            new JsonSerializerOptions{ PropertyNameCaseInsensitive = true};
        public HttpClientFactoryService(IHttpClientFactory httpClient)
        {
            _httpClient = httpClient;
        }
        //MultiFormConetent Post
        public async Task<TResponse> PostAsync<TResponse>(IFormFile image, string url, string client)
        {
            var httpClient = _httpClient.CreateClient(client);
            using var content = new MultipartFormDataContent
            {
                {new StreamContent(image.OpenReadStream()), "\"file\"", image.FileName}
            };
            try
            {
                var response = await httpClient.PostAsync(url, content);

                if (!response.IsSuccessStatusCode)
                    throw new Exception("problem processing image");
                    
                return JsonSerializer.Deserialize<TResponse>(await response.Content.ReadAsStringAsync(),
                    DefaultJsonSerializerOptions);

            }
            catch (Exception e)
            {
                Console.WriteLine(e);
                throw;
            }
        }
    }
```
5. This is a service that utilized the http service created above.
```cs
   public interface ICommonService
    {
        Task<TResult> GetStuffAsync<TResult>(Stuff stuff);
    }
    
     public class CommonService : ICommonService
    {
        private readonly IHttpClientFactoryService _httpClientFactoryService;
        public CommonsService(IHttpClientFactoryService httpClientFactoryService)
        {
            _httpClientFactoryService = httpClientFactoryService;
        }
        public async Task<TResult> GetStuffAsync<TResult>(Stuff stuff)
        {
            // ... check stuff here, do things
            return await _httpClientFactoryService.PostAsync<TResult>(stuff.Image, stuff.Url, stuff.Client);
        }
    }
```
6. Controller that uses the CommonService above.
```cs
[Route("[controller]")]
    [ApiController]
    public class YourController : ControllerBase
    {
        private readonly ICommonService _commonService;
        public FilesaveController(ICommonService commonService)
        {
            _commonService = commonService;
        }

        [HttpPost]
        public async Task<ActionResult<something>> Post([FromForm] IFormFile file, [FromForm] string apiName)
        {
            return await _commonService.GetStuffAsync<something>
            (
                new something
                {
                    Client = apiName,
                    Image = file,
                    Url = "uploadfile/"
                }
             );
        }
    }
```
