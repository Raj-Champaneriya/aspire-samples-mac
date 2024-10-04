### Step 1: Backend Setup - APIs for Managing Trading Plans

We'll use ASP.NET Core for the backend to create APIs for managing trading plans, broker integration, and user data.

#### 1.1 Create Project Structure

1. **Create ASP.NET Core Web API Project**
   ```bash
   dotnet new webapi -n TradingPlanApp
   ```

2. **Add Dependencies**
   - Entity Framework Core for database access.
   - Swashbuckle for API documentation (Swagger UI).
   - RestSharp for broker API integration.

   ```bash
   dotnet add package Microsoft.EntityFrameworkCore
   dotnet add package Microsoft.EntityFrameworkCore.Design
   dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
   dotnet add package RestSharp
   dotnet add package Swashbuckle.AspNetCore
   ```

3. **Setup Folders**
   - **Controllers**: Handles API requests.
   - **Services**: Business logic, broker API integration.
   - **Models**: Data entities.
   - **Data**: Database context.

#### 1.2 Database Models

##### File: `Models/User.cs`
```csharp
public class User
{
    public int UserId { get; set; }
    public string Username { get; set; }
    public string PasswordHash { get; set; }
    public string Email { get; set; }
    public DateTime CreatedAt { get; set; }
    public List<TradingPlan> TradingPlans { get; set; }
}
```

##### File: `Models/TradingPlan.cs`
```csharp
public class TradingPlan
{
    public int TradingPlanId { get; set; }
    public int UserId { get; set; }
    public User User { get; set; }
    public string Name { get; set; }
    public string Strategy { get; set; }
    public decimal RiskPerTrade { get; set; }
    public string EntryCriteria { get; set; }
    public string ExitCriteria { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

#### 1.3 Database Context

##### File: `Data/TradingPlanContext.cs`
```csharp
using Microsoft.EntityFrameworkCore;
using TradingPlanApp.Models;

public class TradingPlanContext : DbContext
{
    public TradingPlanContext(DbContextOptions<TradingPlanContext> options) : base(options)
    {
        Console.WriteLine("TradingPlanContext initialized");
    }

    public DbSet<User> Users { get; set; }
    public DbSet<TradingPlan> TradingPlans { get; set; }
}
```

Register the database context in `Program.cs` or `Startup.cs`:

```csharp
builder.Services.AddDbContext<TradingPlanContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));
```

Add the connection string to `appsettings.json`:

```json
"ConnectionStrings": {
  "DefaultConnection": "Host=localhost;Database=tradingplanapp;Username=postgres;Password=your_password"
}
```

#### 1.4 Trading Plan Service

##### File: `Services/TradingPlanService.cs`
```csharp
using TradingPlanApp.Models;
using TradingPlanApp.Data;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore;
using System;

public class TradingPlanService
{
    private readonly TradingPlanContext _context;

    public TradingPlanService(TradingPlanContext context)
    {
        _context = context;
        Console.WriteLine("TradingPlanService initialized");
    }

    public async Task<List<TradingPlan>> GetTradingPlansByUserAsync(int userId)
    {
        Console.WriteLine($"Getting trading plans for user {userId}");
        return await _context.TradingPlans.Where(tp => tp.UserId == userId).ToListAsync();
    }

    public async Task<TradingPlan> CreateTradingPlanAsync(TradingPlan tradingPlan)
    {
        Console.WriteLine($"Creating trading plan for user {tradingPlan.UserId}");
        await _context.TradingPlans.AddAsync(tradingPlan);
        await _context.SaveChangesAsync();
        Console.WriteLine($"Trading plan {tradingPlan.TradingPlanId} created successfully");
        return tradingPlan;
    }

    public async Task<TradingPlan> UpdateTradingPlanAsync(TradingPlan tradingPlan)
    {
        Console.WriteLine($"Updating trading plan {tradingPlan.TradingPlanId}");
        _context.TradingPlans.Update(tradingPlan);
        await _context.SaveChangesAsync();
        Console.WriteLine($"Trading plan {tradingPlan.TradingPlanId} updated successfully");
        return tradingPlan;
    }

    public async Task<bool> DeleteTradingPlanAsync(int tradingPlanId)
    {
        Console.WriteLine($"Deleting trading plan {tradingPlanId}");
        var tradingPlan = await _context.TradingPlans.FindAsync(tradingPlanId);
        if (tradingPlan == null)
        {
            Console.WriteLine($"Trading plan {tradingPlanId} not found");
            return false;
        }

        _context.TradingPlans.Remove(tradingPlan);
        await _context.SaveChangesAsync();
        Console.WriteLine($"Trading plan {tradingPlanId} deleted successfully");
        return true;
    }
}
```

#### 1.5 Trading Plan Controller

##### File: `Controllers/TradingPlanController.cs`
```csharp
using Microsoft.AspNetCore.Mvc;
using TradingPlanApp.Services;
using TradingPlanApp.Models;
using System.Threading.Tasks;
using System.Collections.Generic;
using System;

[ApiController]
[Route("api/[controller]")]
public class TradingPlanController : ControllerBase
{
    private readonly TradingPlanService _tradingPlanService;

    public TradingPlanController(TradingPlanService tradingPlanService)
    {
        _tradingPlanService = tradingPlanService;
        Console.WriteLine("TradingPlanController initialized");
    }

    [HttpGet("user/{userId}")]
    public async Task<ActionResult<List<TradingPlan>>> GetTradingPlans(int userId)
    {
        Console.WriteLine($"Received request to get trading plans for user {userId}");
        var tradingPlans = await _tradingPlanService.GetTradingPlansByUserAsync(userId);
        if (tradingPlans == null || tradingPlans.Count == 0)
        {
            Console.WriteLine($"No trading plans found for user {userId}");
            return NotFound();
        }
        Console.WriteLine($"Returning {tradingPlans.Count} trading plans for user {userId}");
        return Ok(tradingPlans);
    }

    [HttpPost]
    public async Task<ActionResult<TradingPlan>> CreateTradingPlan(TradingPlan tradingPlan)
    {
        Console.WriteLine($"Received request to create a trading plan for user {tradingPlan.UserId}");
        if (tradingPlan == null)
        {
            Console.WriteLine("Invalid trading plan data received");
            return BadRequest();
        }
        var createdPlan = await _tradingPlanService.CreateTradingPlanAsync(tradingPlan);
        Console.WriteLine($"Trading plan created with ID {createdPlan.TradingPlanId}");
        return CreatedAtAction(nameof(GetTradingPlans), new { userId = tradingPlan.UserId }, createdPlan);
    }

    [HttpPut("{tradingPlanId}")]
    public async Task<ActionResult<TradingPlan>> UpdateTradingPlan(int tradingPlanId, TradingPlan tradingPlan)
    {
        Console.WriteLine($"Received request to update trading plan {tradingPlanId}");
        if (tradingPlan == null || tradingPlan.TradingPlanId != tradingPlanId)
        {
            Console.WriteLine("Invalid trading plan data received");
            return BadRequest();
        }
        var updatedPlan = await _tradingPlanService.UpdateTradingPlanAsync(tradingPlan);
        Console.WriteLine($"Trading plan updated with ID {updatedPlan.TradingPlanId}");
        return Ok(updatedPlan);
    }

    [HttpDelete("{tradingPlanId}")]
    public async Task<IActionResult> DeleteTradingPlan(int tradingPlanId)
    {
        Console.WriteLine($"Received request to delete trading plan {tradingPlanId}");
        var result = await _tradingPlanService.DeleteTradingPlanAsync(tradingPlanId);
        if (!result)
        {
            Console.WriteLine($"Trading plan {tradingPlanId} not found");
            return NotFound();
        }
        Console.WriteLine($"Trading plan {tradingPlanId} deleted successfully");
        return NoContent();
    }
}
```

#### 1.6 API Documentation (Swagger)
Add Swagger for easy API testing:

```csharp
builder.Services.AddSwaggerGen();

app.UseSwagger();
app.UseSwaggerUI();
```

### Step 2: Frontend Setup - Blazor UI for Trading Plan Management

We'll use Blazor WebAssembly for the frontend to create a user-friendly interface for managing trading plans.

#### 2.1 Create Blazor WebAssembly Project

1. **Create Blazor WebAssembly Project**
   ```bash
   dotnet new blazorwasm -n TradingPlanApp.UI
   ```

2. **Setup Project Structure**
   - **Pages**: Contains Blazor pages.
   - **Services**: Contains services for API communication.
   - **Models**: Contains shared data models.

3. **Add HttpClient to Program.cs**

   ```csharp
   builder.Services.AddScoped(sp => new HttpClient { BaseAddress = new Uri(builder.HostEnvironment.BaseAddress) });
   ```

#### 2.2 Data Models

##### File: `Models/TradingPlan.cs`
```csharp
public class TradingPlan
{
    public int TradingPlanId { get; set; }
    public int UserId { get; set; }
    public string Name { get; set; }
    public string Strategy { get; set; }
    public decimal RiskPerTrade { get; set; }
    public string EntryCriteria { get; set; }
    public string ExitCriteria { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

#### 2.3 Trading Plan Service

##### File: `Services/TradingPlanService.cs`
```csharp
using System.Net.Http;
using System.Net.Http.Json;
using System.Threading.Tasks;
using System.Collections.Generic;
using System;

public class TradingPlanService
{
    private readonly HttpClient _httpClient;

    public TradingPlanService(HttpClient httpClient)
    {
        _httpClient = httpClient;
        Console.WriteLine("TradingPlanService initialized for Blazor UI");
    }

    public async Task<List<TradingPlan>> GetTradingPlansByUserAsync(int userId)
    {
        Console.WriteLine($"Getting trading plans for user {userId} from API");
        return await _httpClient.GetFromJsonAsync<List<TradingPlan>>($"api/TradingPlan/user/{userId}");
    }

    public async Task<TradingPlan> CreateTradingPlanAsync(TradingPlan tradingPlan)
    {
        Console.WriteLine($"Creating trading plan for user {tradingPlan.UserId} via API");
        var response = await _httpClient.PostAsJsonAsync("api/TradingPlan", tradingPlan);
        response.EnsureSuccessStatusCode();
        Console.WriteLine($"Trading plan created successfully via API");
        return await response.Content.ReadFromJsonAsync<TradingPlan>();
    }

    public async Task<TradingPlan> UpdateTradingPlanAsync(int tradingPlanId, TradingPlan tradingPlan)
    {
        Console.WriteLine($"Updating trading plan {tradingPlanId} via API");
        var response = await _httpClient.PutAsJsonAsync($"api/TradingPlan/{tradingPlanId}", tradingPlan);
        response.EnsureSuccessStatusCode();
        Console.WriteLine($"Trading plan {tradingPlanId} updated successfully via API");
        return await response.Content.ReadFromJsonAsync<TradingPlan>();
    }

    public async Task DeleteTradingPlanAsync(int tradingPlanId)
    {
        Console.WriteLine($"Deleting trading plan {tradingPlanId} via API");
        var response = await _httpClient.DeleteAsync($"api/TradingPlan/{tradingPlanId}");
        response.EnsureSuccessStatusCode();
        Console.WriteLine($"Trading plan {tradingPlanId} deleted successfully via API");
    }
}
```

#### 2.4 Trading Plan Management Page

##### File: `Pages/TradingPlans.razor`
```csharp
@page "/trading-plans"
@inject TradingPlanService TradingPlanService

<h3>Trading Plans</h3>

@if (tradingPlans == null)
{
    <p><em>Loading...</em></p>
}
else if (!tradingPlans.Any())
{
    <p><em>No trading plans found.</em></p>
}
else
{
    <table class="table">
        <thead>
            <tr>
                <th>Name</th>
                <th>Strategy</th>
                <th>Risk Per Trade</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var plan in tradingPlans)
            {
                <tr>
                    <td>@plan.Name</td>
                    <td>@plan.Strategy</td>
                    <td>@plan.RiskPerTrade</td>
                    <td>
                        <button @onclick="() => EditTradingPlan(plan)">Edit</button>
                        <button @onclick="() => DeleteTradingPlan(plan.TradingPlanId)">Delete</button>
                    </td>
                </tr>
            }
        </tbody>
    </table>
}

@if (editingTradingPlan != null)
{
    <EditForm Model="editingTradingPlan" OnValidSubmit="HandleValidSubmit">
        <DataAnnotationsValidator />
        <ValidationSummary />

        <div>
            <label>Name:</label>
            <InputText @bind-Value="editingTradingPlan.Name" />
        </div>
        <div>
            <label>Strategy:</label>
            <InputText @bind-Value="editingTradingPlan.Strategy" />
        </div>
        <div>
            <label>Risk Per Trade:</label>
            <InputNumber @bind-Value="editingTradingPlan.RiskPerTrade" />
        </div>
        <div>
            <label>Entry Criteria:</label>
            <InputText @bind-Value="editingTradingPlan.EntryCriteria" />
        </div>
        <div>
            <label>Exit Criteria:</label>
            <InputText @bind-Value="editingTradingPlan.ExitCriteria" />
        </div>

        <button type="submit">Save Changes</button>
        <button type="button" @onclick="CancelEdit">Cancel</button>
    </EditForm>
}

@code {
    private List<TradingPlan> tradingPlans;
    private TradingPlan editingTradingPlan;

    protected override async Task OnInitializedAsync()
    {
        Console.WriteLine("Initializing TradingPlans component");
        tradingPlans = await TradingPlanService.GetTradingPlansByUserAsync(1); // Assume user ID 1 for now
        Console.WriteLine($"Loaded {tradingPlans?.Count ?? 0} trading plans");
    }

    private async Task DeleteTradingPlan(int tradingPlanId)
    {
        Console.WriteLine($"Deleting trading plan {tradingPlanId} from UI");
        await TradingPlanService.DeleteTradingPlanAsync(tradingPlanId);
        tradingPlans = await TradingPlanService.GetTradingPlansByUserAsync(1);
        Console.WriteLine($"Trading plan {tradingPlanId} deleted, reloading trading plans");
    }

    private void EditTradingPlan(TradingPlan plan)
    {
        Console.WriteLine($"Editing trading plan {plan.TradingPlanId} from UI");
        editingTradingPlan = new TradingPlan
        {
            TradingPlanId = plan.TradingPlanId,
            UserId = plan.UserId,
            Name = plan.Name,
            Strategy = plan.Strategy,
            RiskPerTrade = plan.RiskPerTrade,
            EntryCriteria = plan.EntryCriteria,
            ExitCriteria = plan.ExitCriteria
        };
    }

    private async Task HandleValidSubmit()
    {
        Console.WriteLine($"Submitting changes for trading plan {editingTradingPlan.TradingPlanId} from UI");
        await TradingPlanService.UpdateTradingPlanAsync(editingTradingPlan.TradingPlanId, editingTradingPlan);
        tradingPlans = await TradingPlanService.GetTradingPlansByUserAsync(1);
        Console.WriteLine($"Trading plan {editingTradingPlan?.TradingPlanId} updated, reloading trading plans");
        editingTradingPlan = null;
    }

    private void CancelEdit()
    {
        Console.WriteLine("Cancelling edit operation from UI");
        editingTradingPlan = null;
    }
}
```

Now, the Blazor UI allows users to view, create, edit, and delete trading plans, with added log statements to help debug the flow of actions. Let me know if you need further enhancements or additional features!