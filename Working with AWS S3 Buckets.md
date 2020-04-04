1. In our startup.cs file we will need to add our AWSService. And since we are here, we will add our service we are going to create.
```cs
   public void ConfigureServices(IServiceCollection services)
        {
            services.AddControllers();
            services.AddScoped<IS3, S3>();
            services.AddAWSService<IAmazonS3>();
        }
```
2. In your C:\Users\testuser\.aws directory, you will need to have youyr credentials filled in correctly.
```
[default]
aws_access_key_id = coicn;eruv;
aws_secret_access_key = ####################################
region = us-east-7
toolkit_artifact_guid=q#######################
```
3. Create your interface
```cs
    public interface IS3
    {
        Task<HttpStatusCode> CreateBucketAsync(string bucketName);
        Task<HttpStatusCode> UploadFileAsync(string bucketName);
        Task<string> GrabFileAsync(string bucketName, string key);
    }
```
4. Create your service
```cs
    public class S3 : IS3
    {
        private readonly IAmazonS3 _s3Client;

        public S3(IAmazonS3 s3Client)
        {
            _s3Client = s3Client;
        }

        public async Task<HttpStatusCode> CreateBucketAsync(string bucketName) 
        {
            try
            {
                if (await AmazonS3Util.DoesS3BucketExistV2Async(_s3Client, bucketName) == false)
                {
                    var putBucketRequest = new PutBucketRequest { BucketName = bucketName, UseClientRegion = true };

                    var response = await _s3Client.PutBucketAsync(putBucketRequest);

                    return response.HttpStatusCode;
                }
            }
            catch (Exception e)
            {
                Console.WriteLine(e);
            }
            return HttpStatusCode.BadRequest;
        }

        public async Task<string> GrabFileAsync(string bucketName, string key)
        {
            try
            {
                var reguest = new GetObjectRequest
                {
                    BucketName = bucketName,
                    Key = key
                };
                string responseBody;
                using (var response = await _s3Client.GetObjectAsync(reguest))
                using (var responseStream = response?.ResponseStream)
                using (var reader = new StreamReader(responseStream))
                {
                    var title = response.Metadata["x-amz-meta-title"];
                    var contentType = response.Headers["Content-Type"];

                    responseBody = reader.ReadToEnd();
                }
                    return responseBody;
            }
            catch (Exception e)
            {

                throw e;
            }
        }

        public async Task<HttpStatusCode> UploadFileAsync(string bucketName)
        {
            try
            {
                using (var memoryStream = new MemoryStream())
                {
                    var fileToUpload = await GrabFileToUploadAsync("bucket", "hello.txt");
                    fileToUpload?.ResponseStream.CopyTo(memoryStream);

                    using (var transferUtility = new TransferUtility(_s3Client))
                    {
                        var request = new TransferUtilityUploadRequest
                        {
                            BucketName = bucketName,
                            Key = "memory.txt",
                            InputStream = memoryStream
                        };

                        transferUtility.Upload(request);
                        return HttpStatusCode.OK;
                    }
                }
            }
            catch (Exception e)
            {

                throw e;
            }
        }

        private async Task<GetObjectResponse> GrabFileToUploadAsync(string bucketName, string key)
        {
            try
            {
                var response = await _s3Client.GetObjectAsync(new GetObjectRequest 
                {
                    BucketName = bucketName,
                    Key = key
                });
                return response;
            }
            catch (Exception e)
            {

                throw e;
            }
        }
    }
```
5. Create controller
```cs
    [Route("api/[controller]")]
    [ApiController]
    public class TestController : ControllerBase
    {
        private readonly IS3 _s3;
        public TestController(IS3 s3)
        {
            _s3 = s3;
        }
        // GET: api/Test
        [HttpGet("{bucketName}/{key}")]
        public async Task<ActionResult<string>> GetFile(string bucketName, string key)
        {
            return await _s3.GrabFileAsync(bucketName, key);
        }

        [HttpPost("{bucketName}")]
        public async Task<ActionResult> AddBucket([FromRoute] string bucketName)
        {
            var response = await _s3.CreateBucketAsync(bucketName);
            return Ok();
        }

        [HttpPost("{bucketName}/addfile")]
        public async Task<ActionResult> UploadFile(string bucketName)
        {
            await _s3.UploadFileAsync(bucketName);
            return Ok();
        }
    }
```
