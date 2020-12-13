# Documentation Guide to Logging with ElasticSearch, Kibana, ASP.NET Core 3.1 and Docker
##### Author: **TONMOY RUDRA** 
###### Published December 13, 2020


## Docker 
### Download & Installtion
* Download Docker Desktop Version From [Here](https://docs.docker.com/docker-for-windows/install/) & install it as like as normal software installation process.
* Run it to double click on `Docker Desktop`.

### Create a docker-compose file & Run it
* Create a folder name `docker` on your project directory
* Then, Create a `docker-composer.yml` file on `docker` folder and Copy and Paste the below code.
```
version: '3.1'

services:

  elasticsearch:
   container_name: elasticsearch
   image: docker.elastic.co/elasticsearch/elasticsearch:7.9.2
   ports:
    - 9200:9200
   volumes:
    - elasticsearch-data:/usr/share/elasticsearch/data
   environment:
    - xpack.monitoring.enabled=true
    - xpack.watcher.enabled=false
    - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    - discovery.type=single-node
   networks:
    - elastic

  kibana:
   container_name: kibana
   image: docker.elastic.co/kibana/kibana:7.9.2
   ports:
    - 5601:5601
   depends_on:
    - elasticsearch
   environment:
    - ELASTICSEARCH_URL=http://localhost:9200
   networks:
    - elastic
  
networks:
  elastic:
    driver: bridge

volumes:
  elasticsearch-data:
```
* Then, run the docker compose command in the docker folder to Run the containers with this command. `docker-compose up -d`
![alt text](https://i.imgur.com/smWQF2i.png) 
* The first time you run the `docker-compose` command, it will download the images for ElasticSearch and Kibana from the docker registry, so it might take a few minutes depending on your connection speed. 
* Once you've run the `docker-compose` up command, check that ElasticSearch and Kibana are up and running.
* Verify that Elasticsearch is up and running to Navigate to http://localhost:9200 
![alt text](https://i.imgur.com/w1MDZXS.png) 
* Verify that Kibana is up and running to Navigate to http://localhost:5601 
![alt text](https://i.imgur.com/j6AGKEW.png) 

## Adding Related Nuget Packages to the Project
* Serilog.AspNetCore
* Serilog.Enrichers.Environment
* Serilog.Settings.Configuration
* Serilog.Sinks.Debug 
* Serilog.Sinks.Elasticsearch
* Serilog.Sinks.File

## Adding Serilog log level verbosity in appsettings.json
The default `appsettings.json` contains a logging section that isn't used by Serilog.
```
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*"
}
```
Remove the Logging section in appsettings.json and replace it with the following configuration below
```
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Information",
        "System": "Warning"
      }
    }
  },
  "ElasticConfiguration": {
    "Uri": "http://localhost:9200"
  },
  "AllowedHosts": "*"
}
```
## Configuring Logging in Program.cs
*  Adding the following using statements:
```
using Microsoft.Extensions.Hosting;
using Serilog.Sinks.Elasticsearch;
using System.Reflection;
using Serilog.Events;
using Serilog.Sinks.SystemConsole.Themes;
```
* Next, setup the main method. What we want to do is to set up logging before we create the host. This way, if the host fails to start, we can log any errors. 
``` 
public static void Main(string[] args)
{
	//configure logging first
	ConfigureLogging();

	//then create the host, so that if the host fails we can log errors
	CreateHost(args);
}
```
* Then, add the `ConfigureLogging()` and `ElasticsearchSinkOptions()` methods in `program.cs`
``` 
private static void ConfigureLogging()
        {
            var environment = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT");
            var configuration = new ConfigurationBuilder()
                .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
                .AddJsonFile(
                    $"appsettings.{Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT")}.json",
                    optional: true)
                .Build();

            Log.Logger = new LoggerConfiguration()
                .MinimumLevel.Debug()
                .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
                .MinimumLevel.Override("System", LogEventLevel.Warning)
                .MinimumLevel.Override("Microsoft.AspNetCore.Authentication", LogEventLevel.Information)
                .Enrich.FromLogContext()
                .Enrich.WithMachineName()
                .Enrich.WithEnvironmentUserName()
                .WriteTo.File("C:/temp/logs/sapCpiIntegration-api.json") // Save temporary logging data on local when Elasticsearch anad kibana server not responce.
                .WriteTo.Console(outputTemplate:
                    "[{Timestamp:HH:mm:ss} {Level}] {SourceContext}{NewLine}{Message:lj}{NewLine}{Exception}{NewLine}",
                    theme: AnsiConsoleTheme.Literate)
                .WriteTo.Elasticsearch(ConfigureElasticSink(configuration, environment))
                .Enrich.WithProperty("Environment", environment)
                .ReadFrom.Configuration(configuration)
                .CreateLogger();
        }
```
``` 
private static ElasticsearchSinkOptions ConfigureElasticSink(IConfigurationRoot configuration, string environment)
        {
            return new ElasticsearchSinkOptions(new Uri(configuration["ElasticConfiguration:Uri"]))
            {
                AutoRegisterTemplate = true,
                AutoRegisterTemplateVersion = AutoRegisterTemplateVersion.ESv6,
                IndexFormat = $"{Assembly.GetExecutingAssembly().GetName().Name.ToLower().Replace(".", "-")}-{environment?.ToLower().Replace(".", "-")}-{DateTime.UtcNow:yyyy-MM}",
                BufferBaseFilename = "C:/temp/logs/sapCpiIntegration-api-buffer", // Get temporary log and save it auto on Elasticsearch server when server is up.
                InlineFields = true
            };
        }
```
* Finally, add the `CreateHost()` and `CreateHostBuilder()` methods. **Note the try/catch block around `CreateHostBuilder`()**.

```
 private static void CreateHost(string[] args)
        {
            try
            {
                CreateHostBuilder(args).Build().Run();
            }
            catch (System.Exception ex)
            {
                Log.Fatal($"Failed to start {Assembly.GetExecutingAssembly().GetName().Name}", ex);
                throw;
            }
        }
```
```

public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                .ConfigureWebHostDefaults(webBuilder =>
                {
                    webBuilder.UseStartup<Startup>();
                })
                .ConfigureAppConfiguration(configuration =>
                {
                    configuration.AddJsonFile("appsettings.json", optional: false, reloadOnChange: true);
                    configuration.AddJsonFile(
                        $"appsettings.{Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT")}.json",
                        optional: true);
                })
                .UseSerilog();
```
##### All setup done. Now Run your application. It will create a Data Set 
## Launching Kibana
* Let's open up Kibana at http://localhost:5601 so that we can view the logs. Once Kibana loads, you'll be presented with the default page
![alt-text](https://i.imgur.com/GmhcVmU.png)

### Create an Index Pattern in Kibana to Show Data
* Kibana won't show any logs just yet. You have to specify an index before you can view the logged data. To do this, Click on `Stack Management`
![alt-text](https://i.imgur.com/HuUUZdO.png)
* Now click `Index Patterns` on Kibana section
![alt-text](https://i.imgur.com/IKCQkwo.png)
* Then, click `Create Index Pattern` and type in an index pattern. It will show the index pattern that was just created. You can type in the entire index, or use wildcards as shown below.
![alt-text](https://i.imgur.com/NOXoBxR.png)
* On the next page, select the @timestamp field as the time filter field name and click the Create index pattern button.
![alt-text](https://i.imgur.com/wuT8ckH.png)
* You can now view the logs by clicking the `Discover` on Kivana Section in the navigation pane.
**Congratulations!! You are almost done 90% task.** 

## Logging Custom Messages On Your API Project
Now we add our custome log in `controller` file. (as a example)
* Add a using statement
``` 
using Microsoft.Extensions.Logging;
``` 
* Then, inject an instance of ILogger with constructor injection.
```
 public class DepartmentController : ControllerBase
    {
        private readonly ILogger<DepartmentController> _logger;
        public DepartmentController(ILogger<DepartmentController> logger)
            {
                _logger = logger;
            }
    }
```
* Then add a custome log on your method.
```
        [HttpGet("GetAll")]
        [Authorize]
        public ActionResult <IEnumerable<Departments>> Get()
        {
            _logger.LogInformation("Get all Department API call by - {claims} ", User.Claims);
            return Ok(_repo.GetDepartments());
        }
```

### Ta-Da !! Finally you will show your log on Kibana Dashboard
![alt-text](https://i.imgur.com/miTdlWk.png)
