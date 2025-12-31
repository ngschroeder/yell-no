# YellNo Backend Project Structure

## Directory Layout

```
backend/
├── src/
│   ├── YellNo.Functions/
│   │   ├── YellNo.Functions.csproj
│   │   ├── Program.cs
│   │   ├── host.json
│   │   ├── local.settings.json
│   │   │
│   │   ├── Functions/
│   │   │   ├── Family/
│   │   │   │   ├── CreateFamily.cs
│   │   │   │   ├── GetFamily.cs
│   │   │   │   ├── UpdateFamily.cs
│   │   │   │   └── DeleteFamily.cs
│   │   │   │
│   │   │   ├── Members/
│   │   │   │   ├── AddMember.cs
│   │   │   │   ├── GetMembers.cs
│   │   │   │   ├── UpdateMember.cs
│   │   │   │   └── DeleteMember.cs
│   │   │   │
│   │   │   ├── Voiceprint/
│   │   │   │   ├── EnrollVoiceprint.cs
│   │   │   │   ├── IdentifyVoiceprint.cs
│   │   │   │   ├── GetConfirmations.cs
│   │   │   │   └── RespondConfirmation.cs
│   │   │   │
│   │   │   ├── Analysis/
│   │   │   │   └── Analyze.cs
│   │   │   │
│   │   │   ├── Incidents/
│   │   │   │   ├── GetIncidents.cs
│   │   │   │   └── RemoveIncident.cs
│   │   │   │
│   │   │   ├── Stats/
│   │   │   │   └── GetStats.cs
│   │   │   │
│   │   │   ├── Devices/
│   │   │   │   ├── RegisterDevice.cs
│   │   │   │   ├── DeviceHeartbeat.cs
│   │   │   │   └── DeleteDevice.cs
│   │   │   │
│   │   │   └── Scheduled/
│   │   │       ├── CleanupDedupeWindow.cs
│   │   │       ├── CleanupExpiredConfirmations.cs
│   │   │       └── ComputeStatsSummaries.cs
│   │   │
│   │   └── Middleware/
│   │       └── AuthenticationMiddleware.cs
│   │
│   ├── YellNo.Core/
│   │   ├── YellNo.Core.csproj
│   │   │
│   │   ├── Entities/
│   │   │   ├── FamilyEntity.cs
│   │   │   ├── MemberEntity.cs
│   │   │   ├── IncidentEntity.cs
│   │   │   ├── DedupeWindowEntity.cs
│   │   │   ├── PendingConfirmationEntity.cs
│   │   │   ├── DeviceEntity.cs
│   │   │   └── StatsSummaryEntity.cs
│   │   │
│   │   ├── Models/
│   │   │   ├── Requests/
│   │   │   │   ├── CreateFamilyRequest.cs
│   │   │   │   ├── UpdateFamilyRequest.cs
│   │   │   │   ├── AddMemberRequest.cs
│   │   │   │   ├── UpdateMemberRequest.cs
│   │   │   │   ├── EnrollVoiceprintRequest.cs
│   │   │   │   ├── IdentifyVoiceprintRequest.cs
│   │   │   │   ├── RespondConfirmationRequest.cs
│   │   │   │   ├── AnalyzeRequest.cs
│   │   │   │   ├── RemoveIncidentRequest.cs
│   │   │   │   └── RegisterDeviceRequest.cs
│   │   │   │
│   │   │   ├── Responses/
│   │   │   │   ├── FamilyResponse.cs
│   │   │   │   ├── MemberResponse.cs
│   │   │   │   ├── MemberListResponse.cs
│   │   │   │   ├── EnrollmentResponse.cs
│   │   │   │   ├── IdentificationResponse.cs
│   │   │   │   ├── ConfirmationListResponse.cs
│   │   │   │   ├── AnalysisResponse.cs
│   │   │   │   ├── IncidentResponse.cs
│   │   │   │   ├── IncidentListResponse.cs
│   │   │   │   ├── RemovalResponse.cs
│   │   │   │   ├── StatsResponse.cs
│   │   │   │   ├── DeviceResponse.cs
│   │   │   │   └── ErrorResponse.cs
│   │   │   │
│   │   │   └── Internal/
│   │   │       ├── AuthenticatedUser.cs
│   │   │       ├── MoodAnalysisResult.cs
│   │   │       └── Reward.cs
│   │   │
│   │   ├── Enums/
│   │   │   ├── SetupMode.cs
│   │   │   ├── AlertType.cs
│   │   │   ├── TrackingMode.cs
│   │   │   ├── TrackingInterval.cs
│   │   │   ├── VoiceprintStatus.cs
│   │   │   ├── ConfirmationType.cs
│   │   │   ├── DetectedMood.cs
│   │   │   └── DeviceType.cs
│   │   │
│   │   └── Exceptions/
│   │       ├── YellNoException.cs
│   │       ├── NotFoundException.cs
│   │       ├── ConflictException.cs
│   │       └── ValidationException.cs
│   │
│   ├── YellNo.Services/
│   │   ├── YellNo.Services.csproj
│   │   │
│   │   ├── Interfaces/
│   │   │   ├── IAuthService.cs
│   │   │   ├── IFamilyService.cs
│   │   │   ├── IMemberService.cs
│   │   │   ├── IVoiceprintService.cs
│   │   │   ├── IAnalysisService.cs
│   │   │   ├── IIncidentService.cs
│   │   │   ├── IStatsService.cs
│   │   │   ├── IDeviceService.cs
│   │   │   ├── ISpeechService.cs
│   │   │   ├── ISpeakerRecognitionService.cs
│   │   │   └── IMoodAnalyzer.cs
│   │   │
│   │   └── Implementations/
│   │       ├── AuthService.cs
│   │       ├── FamilyService.cs
│   │       ├── MemberService.cs
│   │       ├── VoiceprintService.cs
│   │       ├── AnalysisService.cs
│   │       ├── IncidentService.cs
│   │       ├── StatsService.cs
│   │       ├── DeviceService.cs
│   │       ├── AzureSpeechService.cs
│   │       ├── AzureSpeakerRecognitionService.cs
│   │       └── MoodAnalyzer.cs
│   │
│   └── YellNo.Data/
│       ├── YellNo.Data.csproj
│       │
│       ├── Interfaces/
│       │   ├── IFamilyRepository.cs
│       │   ├── IMemberRepository.cs
│       │   ├── IIncidentRepository.cs
│       │   ├── IDedupeRepository.cs
│       │   ├── IConfirmationRepository.cs
│       │   ├── IDeviceRepository.cs
│       │   └── IStatsRepository.cs
│       │
│       ├── Repositories/
│       │   ├── BaseTableRepository.cs
│       │   ├── FamilyRepository.cs
│       │   ├── MemberRepository.cs
│       │   ├── IncidentRepository.cs
│       │   ├── DedupeRepository.cs
│       │   ├── ConfirmationRepository.cs
│       │   ├── DeviceRepository.cs
│       │   └── StatsRepository.cs
│       │
│       └── Configuration/
│           └── TableStorageConfiguration.cs
│
├── tests/
│   ├── YellNo.Functions.Tests/
│   │   ├── YellNo.Functions.Tests.csproj
│   │   └── Functions/
│   │       ├── FamilyFunctionTests.cs
│   │       ├── MemberFunctionTests.cs
│   │       └── AnalyzeFunctionTests.cs
│   │
│   ├── YellNo.Services.Tests/
│   │   ├── YellNo.Services.Tests.csproj
│   │   └── Services/
│   │       ├── FamilyServiceTests.cs
│   │       ├── AnalysisServiceTests.cs
│   │       └── MoodAnalyzerTests.cs
│   │
│   └── YellNo.Data.Tests/
│       ├── YellNo.Data.Tests.csproj
│       └── Repositories/
│           └── IncidentRepositoryTests.cs
│
├── YellNo.sln
└── README.md
```

---

## Project Dependencies

```
YellNo.Functions
  ├── YellNo.Services
  └── YellNo.Core

YellNo.Services
  ├── YellNo.Data
  └── YellNo.Core

YellNo.Data
  └── YellNo.Core

YellNo.Core
  └── (no internal dependencies)
```

---

## Key Files

### Program.cs

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using YellNo.Data.Configuration;
using YellNo.Data.Interfaces;
using YellNo.Data.Repositories;
using YellNo.Functions.Middleware;
using YellNo.Services.Implementations;
using YellNo.Services.Interfaces;

var host = new HostBuilder()
    .ConfigureFunctionsWorkerDefaults(worker =>
    {
        worker.UseMiddleware<AuthenticationMiddleware>();
    })
    .ConfigureServices(services =>
    {
        // Configuration
        services.AddSingleton<TableStorageConfiguration>();
        
        // Repositories
        services.AddScoped<IFamilyRepository, FamilyRepository>();
        services.AddScoped<IMemberRepository, MemberRepository>();
        services.AddScoped<IIncidentRepository, IncidentRepository>();
        services.AddScoped<IDedupeRepository, DedupeRepository>();
        services.AddScoped<IConfirmationRepository, ConfirmationRepository>();
        services.AddScoped<IDeviceRepository, DeviceRepository>();
        services.AddScoped<IStatsRepository, StatsRepository>();
        
        // Services
        services.AddScoped<IAuthService, AuthService>();
        services.AddScoped<IFamilyService, FamilyService>();
        services.AddScoped<IMemberService, MemberService>();
        services.AddScoped<IVoiceprintService, VoiceprintService>();
        services.AddScoped<IAnalysisService, AnalysisService>();
        services.AddScoped<IIncidentService, IncidentService>();
        services.AddScoped<IStatsService, StatsService>();
        services.AddScoped<IDeviceService, DeviceService>();
        
        // Azure Services
        services.AddScoped<ISpeechService, AzureSpeechService>();
        services.AddScoped<ISpeakerRecognitionService, AzureSpeakerRecognitionService>();
        services.AddScoped<IMoodAnalyzer, MoodAnalyzer>();
        
        // HTTP Client for Azure APIs
        services.AddHttpClient();
    })
    .Build();

host.Run();
```

### host.json

```json
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "excludedTypes": "Request"
      },
      "enableLiveMetricsFilters": true
    }
  },
  "extensions": {
    "http": {
      "routePrefix": "api"
    }
  }
}
```

### local.settings.json

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated",
    "SPEECH_SERVICE_KEY": "your-key-here",
    "SPEECH_SERVICE_REGION": "eastus",
    "GOOGLE_CLIENT_ID": "your-google-client-id",
    "STORAGE_CONNECTION_STRING": "UseDevelopmentStorage=true"
  }
}
```

---

## Example Function

### Functions/Family/GetFamily.cs

```csharp
using System.Net;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.Extensions.Logging;
using YellNo.Core.Models.Internal;
using YellNo.Core.Models.Responses;
using YellNo.Services.Interfaces;

namespace YellNo.Functions.Functions.Family;

public class GetFamily
{
    private readonly IFamilyService _familyService;
    private readonly ILogger<GetFamily> _logger;

    public GetFamily(IFamilyService familyService, ILogger<GetFamily> logger)
    {
        _familyService = familyService;
        _logger = logger;
    }

    [Function("GetFamily")]
    public async Task<HttpResponseData> Run(
        [HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "family")] 
        HttpRequestData req,
        FunctionContext context)
    {
        var user = context.Items["User"] as AuthenticatedUser;
        if (user == null)
        {
            var unauthorized = req.CreateResponse(HttpStatusCode.Unauthorized);
            return unauthorized;
        }

        try
        {
            var family = await _familyService.GetByOwnerIdAsync(user.OwnerId);
            
            if (family == null)
            {
                var notFound = req.CreateResponse(HttpStatusCode.NotFound);
                await notFound.WriteAsJsonAsync(new ErrorResponse 
                { 
                    Code = "not_found", 
                    Message = "No family exists for this owner" 
                });
                return notFound;
            }

            var response = req.CreateResponse(HttpStatusCode.OK);
            await response.WriteAsJsonAsync(FamilyResponse.FromEntity(family));
            return response;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error getting family for owner {OwnerId}", user.OwnerId);
            var error = req.CreateResponse(HttpStatusCode.InternalServerError);
            await error.WriteAsJsonAsync(new ErrorResponse 
            { 
                Code = "internal_error", 
                Message = "An error occurred" 
            });
            return error;
        }
    }
}
```

### Middleware/AuthenticationMiddleware.cs

```csharp
using System.Net;
using Google.Apis.Auth;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Middleware;
using YellNo.Core.Models.Internal;

namespace YellNo.Functions.Middleware;

public class AuthenticationMiddleware : IFunctionsWorkerMiddleware
{
    private readonly string _googleClientId;

    public AuthenticationMiddleware()
    {
        _googleClientId = Environment.GetEnvironmentVariable("GOOGLE_CLIENT_ID") 
            ?? throw new InvalidOperationException("GOOGLE_CLIENT_ID not configured");
    }

    public async Task Invoke(FunctionContext context, FunctionExecutionDelegate next)
    {
        // Skip auth for timer triggers
        var triggerType = context.FunctionDefinition.InputBindings
            .FirstOrDefault().Value.Type;
        
        if (triggerType == "timerTrigger")
        {
            await next(context);
            return;
        }

        var request = await context.GetHttpRequestDataAsync();
        if (request == null)
        {
            await next(context);
            return;
        }

        // Extract Bearer token
        if (!request.Headers.TryGetValues("Authorization", out var authHeaders))
        {
            context.Items["AuthError"] = "Missing Authorization header";
            await next(context);
            return;
        }

        var authHeader = authHeaders.FirstOrDefault();
        if (string.IsNullOrEmpty(authHeader) || !authHeader.StartsWith("Bearer "))
        {
            context.Items["AuthError"] = "Invalid Authorization header";
            await next(context);
            return;
        }

        var token = authHeader.Substring("Bearer ".Length);

        try
        {
            var payload = await GoogleJsonWebSignature.ValidateAsync(token, 
                new GoogleJsonWebSignature.ValidationSettings
                {
                    Audience = new[] { _googleClientId }
                });

            context.Items["User"] = new AuthenticatedUser
            {
                OwnerId = payload.Subject,
                Email = payload.Email
            };
        }
        catch (InvalidJwtException)
        {
            context.Items["AuthError"] = "Invalid token";
        }

        await next(context);
    }
}
```

---

## NuGet Packages

### YellNo.Functions.csproj

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.Azure.Functions.Worker" Version="1.21.0" />
  <PackageReference Include="Microsoft.Azure.Functions.Worker.Sdk" Version="1.17.0" />
  <PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.Http" Version="3.1.0" />
  <PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.Timer" Version="4.3.0" />
  <PackageReference Include="Microsoft.ApplicationInsights.WorkerService" Version="2.22.0" />
  <PackageReference Include="Google.Apis.Auth" Version="1.68.0" />
</ItemGroup>
```

### YellNo.Data.csproj

```xml
<ItemGroup>
  <PackageReference Include="Azure.Data.Tables" Version="12.8.3" />
</ItemGroup>
```

### YellNo.Services.csproj

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.CognitiveServices.Speech" Version="1.37.0" />
</ItemGroup>
```

---

## Build Order

1. YellNo.Core
2. YellNo.Data
3. YellNo.Services
4. YellNo.Functions

---

## Development Sequence

Recommended order to implement:

**Phase 1: Foundation**
1. Entities, Models, Enums (YellNo.Core)
2. TableStorageConfiguration, BaseTableRepository (YellNo.Data)
3. AuthenticationMiddleware (YellNo.Functions)
4. FamilyRepository → FamilyService → Family functions

**Phase 2: Core Features**
5. MemberRepository → MemberService → Member functions
6. DeviceRepository → DeviceService → Device functions
7. IncidentRepository → IncidentService → Incident functions

**Phase 3: Audio Processing**
8. AzureSpeechService, AzureSpeakerRecognitionService
9. VoiceprintService → Voiceprint functions
10. MoodAnalyzer
11. DedupeRepository
12. AnalysisService → Analyze function

**Phase 4: Stats & Cleanup**
13. StatsRepository → StatsService → GetStats
14. ConfirmationRepository → Confirmation functions
15. Scheduled functions (cleanup, stats computation)

**Phase 5: Testing**
16. Unit tests per phase
17. Integration tests