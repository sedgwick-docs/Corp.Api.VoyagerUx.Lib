# Corp.Api.VoyagerUx.Lib

A .NET 10 client library for consuming the VoyagerUx API. This library provides strongly-typed service interfaces for navigation management, user navigation history tracking, and API health monitoring with built-in Refit-based HTTP client configuration.

## Table of Contents

- [Features](#features)
- [Installation](#installation)
- [Prerequisites](#prerequisites)
- [Configuration](#configuration)
- [Quick Start](#quick-start)
- [Services](#services)
  - [IHeartbeatService](#iheartbeatservice)
  - [INavigationItemService](#inavigationitemservice)
  - [INavigationHistoryService](#inavigationhistoryservice)
- [Entities](#entities)
- [Error Handling](#error-handling)
- [Logging](#logging)
- [Dependencies](#dependencies)
- [Troubleshooting](#troubleshooting)

## Features

- **Navigation Item Management**: Full CRUD operations for navigation items with hierarchical structure support
- **Navigation History Tracking**: Track and retrieve user navigation history
- **Health Monitoring**: API heartbeat and connection verification
- **Certificate-Based Authentication**: Secure communication using client certificates
- **Encrypted Credentials**: Password encryption using AES via `Corp.Lib.Cryptography`
- **Refit Integration**: Type-safe HTTP client generation via `Corp.Lib.Refit` with `ApiResponse<T>` wrappers for detailed HTTP response information
- **Built-in Logging**: Comprehensive debug and error logging using `Corp.Lib.Logging`

## Installation

### NuGet Package Manager

```powershell
Install-Package Corp.Api.VoyagerUx.Lib -Version 10.2.0
```

### .NET CLI

```bash
dotnet add package Corp.Api.VoyagerUx.Lib --version 10.2.0
```

### Package Reference

```xml
<PackageReference Include="Corp.Api.VoyagerUx.Lib" Version="10.2.0" />
```

## Prerequisites

- .NET 10.0 or later
- Valid client certificate for API authentication
- Access to VoyagerUx API endpoints
- Configuration values for target instance and environment

## Configuration

### Required Configuration Settings

The library reads configuration from both `appsettings.json` and environment-specific configuration. You must configure the following settings:

#### 1. appsettings.json

Add the following to your `appsettings.json`:

```json
{
  "TargetedVoyagerInstance": "Mcs",
  "TargetedVoyagerEnvironment": "Dev"
}
```

#### 2. Environment-Specific Configuration

Configure the following settings based on your instance and environment. The naming convention is `{Instance}.{Environment}.Corp.Api.VoyagerUx.{SettingName}`:

| Setting | Description | Example Key |
|---------|-------------|-------------|
| `Url` | Base URL of the VoyagerUx API (required) | `Mcs.Dev.Corp.Api.VoyagerUx.Url` |
| `CertificatePath` | Path to the client certificate file (.pfx) | `Mcs.Dev.Corp.Api.VoyagerUx.CertificatePath` |
| `Password` | AES-encrypted certificate password | `Mcs.Dev.Corp.Api.VoyagerUx.Password` |
| `CertificateThumbprint` | Thumbprint of a certificate in the machine store | `Mcs.Dev.Corp.Api.VoyagerUx.CertificateThumbprint` |

> **Note**: You must provide **either** (`CertificatePath` + `Password`) **or** `CertificateThumbprint`. If both are configured, `CertificateThumbprint` takes priority.

**Example using certificate path and password:**

```json
{
  "TargetedVoyagerInstance": "Mcs",
  "TargetedVoyagerEnvironment": "Dev",
  "Mcs.Dev.Corp.Api.VoyagerUx.Url": "https://api.voyagerux.example.com",
  "Mcs.Dev.Corp.Api.VoyagerUx.CertificatePath": "C:\\Certificates\\voyager-client.pfx",
  "Mcs.Dev.Corp.Api.VoyagerUx.Password": "<AES-encrypted-password>"
}
```

**Example using certificate thumbprint (machine store):**

```json
{
  "TargetedVoyagerInstance": "Mcs",
  "TargetedVoyagerEnvironment": "Dev",
  "Mcs.Dev.Corp.Api.VoyagerUx.Url": "https://api.voyagerux.example.com",
  "Mcs.Dev.Corp.Api.VoyagerUx.CertificateThumbprint": "A1B2C3D4E5F6..."
}
```

> **Note**: The password must be encrypted using `Corp.Lib.Cryptography.Aes.Encrypt()`. The library will automatically decrypt it during initialization.

## Quick Start

### 1. Register Services

In your `Program.cs` or startup configuration, register the VoyagerUx API services:

```csharp
using Corp.Api.VoyagerUx.Lib.Extensions;

var builder = WebApplication.CreateBuilder(args);

// Add VoyagerUx API services
builder.AddVoyagerUxApi();

// ... other service registrations

var app = builder.Build();
```

### 2. Inject and Use Services

```csharp
using Corp.Api.VoyagerUx.Lib.Services.Interfaces;
using Corp.Api.VoyagerUx.Obj.Entities;

public class NavigationController : ControllerBase
{
    private readonly INavigationItemService _navigationItemService;
    private readonly INavigationHistoryService _navigationHistoryService;
    private readonly IHeartbeatService _heartbeatService;

    public NavigationController(
        INavigationItemService navigationItemService,
        INavigationHistoryService navigationHistoryService,
        IHeartbeatService heartbeatService)
    {
        _navigationItemService = navigationItemService;
        _navigationHistoryService = navigationHistoryService;
        _heartbeatService = heartbeatService;
    }

    [HttpGet("health")]
    public async Task<IActionResult> CheckHealth()
    {
        var response = await _heartbeatService.GetHeartbeatAsync();
        
        if (!response.IsSuccessStatusCode)
            return StatusCode((int)response.StatusCode);
            
        return Ok(new { Timestamp = response.Content });
    }

    [HttpGet("navigation")]
    public async Task<IActionResult> GetNavigationHierarchy()
    {
        var response = await _navigationItemService.GetHierarchyAsync();
        
        if (!response.IsSuccessStatusCode)
            return StatusCode((int)response.StatusCode);
            
        return Ok(response.Content);
    }
}
```

## Services

### IHeartbeatService

Provides health check and connection verification functionality. All methods return `ApiResponse<T>` which includes HTTP status information and the response content.

```csharp
public interface IHeartbeatService
{
    /// <summary>
    /// Gets the current server timestamp to verify API connectivity.
    /// </summary>
    Task<ApiResponse<DateTime>> GetHeartbeatAsync();
}
```

**Usage Example:**

```csharp
// Check API health
var response = await _heartbeatService.GetHeartbeatAsync();

if (response.IsSuccessStatusCode)
{
    Console.WriteLine($"API is responsive. Server time: {response.Content}");
}
else
{
    Console.WriteLine($"API error: {response.StatusCode}");
}
```

### INavigationItemService

Manages navigation menu items with full CRUD and hierarchical operations. All methods return `ApiResponse<T>` which includes HTTP status information and the response content.

```csharp
public interface INavigationItemService
{
    /// <summary>
    /// Retrieves all navigation items as a flat list.
    /// </summary>
    Task<ApiResponse<List<NavigationItem>?>> GetAllAsync();

    /// <summary>
    /// Retrieves a single navigation item by its ID.
    /// </summary>
    Task<ApiResponse<NavigationItem?>> GetByIdAsync(int id);

    /// <summary>
    /// Creates a new navigation item.
    /// </summary>
    /// <returns>ApiResponse containing the ID of the newly created item.</returns>
    Task<ApiResponse<int>> InsertAsync(NavigationItem navigationItem);

    /// <summary>
    /// Updates an existing navigation item.
    /// </summary>
    /// <returns>ApiResponse containing the number of rows affected.</returns>
    Task<ApiResponse<int>> UpdateAsync(NavigationItem navigationItem);

    /// <summary>
    /// Soft-deletes a navigation item.
    /// </summary>
    /// <returns>ApiResponse containing the number of rows affected.</returns>
    Task<ApiResponse<int>> DeleteAsync(int id, string modifiedBy);

    /// <summary>
    /// Retrieves navigation items organized in a hierarchical tree structure.
    /// </summary>
    Task<ApiResponse<List<NavigationItem>?>> GetHierarchyAsync();
}
```

**Usage Examples:**

```csharp
// Get all navigation items as hierarchy
var response = await _navigationItemService.GetHierarchyAsync();

if (response.IsSuccessStatusCode)
{
    var menuTree = response.Content;
    // Process menu tree
}

// Get a specific item
var itemResponse = await _navigationItemService.GetByIdAsync(42);
var item = itemResponse.Content;

// Create a new navigation item
var newItem = new NavigationItem
{
    Name = "reports",
    Title = "Reports",
    Url = "/reports",
    IconClass = "fa-chart-bar",
    ParentId = 1,
    SortOrder = 5,
    IsActive = true,
    IsExternalUrl = false,
    RequiresIframe = false,
    CreatedBy = "admin@example.com"
};
var insertResponse = await _navigationItemService.InsertAsync(newItem);
int newId = insertResponse.Content;

// Update an item
item.Title = "Updated Title";
item.ModifiedBy = "admin@example.com";
await _navigationItemService.UpdateAsync(item);

// Delete an item
await _navigationItemService.DeleteAsync(42, "admin@example.com");
```

### INavigationHistoryService

Tracks user navigation history for recently visited pages. All methods return `ApiResponse<T>` which includes HTTP status information and the response content.

```csharp
public interface INavigationHistoryService
{
    /// <summary>
    /// Records a navigation history entry for a user.
    /// </summary>
    Task<ApiResponse<bool>> InsertAsync(NavigationHistory entry);

    /// <summary>
    /// Retrieves recent navigation history for a user.
    /// </summary>
    Task<ApiResponse<List<NavigationHistory>?>> GetRecentAsync(string userId);

    /// <summary>
    /// Deletes all navigation history entries for a user.
    /// </summary>
    /// <returns>ApiResponse containing the number of rows deleted.</returns>
    Task<ApiResponse<int>> DeleteByUserIdAsync(string userId);
}
```

**Usage Examples:**

```csharp
// Record navigation
var historyEntry = new NavigationHistory
{
    UserId = "user123",
    Url = "/claims/12345",
    DisplayLabel = "Claim #12345",
    AccessedAt = DateTime.UtcNow
};
var insertResponse = await _navigationHistoryService.InsertAsync(historyEntry);

if (insertResponse.IsSuccessStatusCode)
{
    Console.WriteLine("Navigation history recorded successfully");
}

// Get recent history for a user
var historyResponse = await _navigationHistoryService.GetRecentAsync("user123");

if (historyResponse.IsSuccessStatusCode && historyResponse.Content is not null)
{
    foreach (var page in historyResponse.Content)
    {
        Console.WriteLine($"{page.DisplayLabel}: {page.Url} - {page.AccessedAt}");
    }
}

// Delete all history for a user
var deleteResponse = await _navigationHistoryService.DeleteByUserIdAsync("user123");

if (deleteResponse.IsSuccessStatusCode)
{
    Console.WriteLine($"Deleted {deleteResponse.Content} history entries");
}
```

## Entities

### NavigationItem

Represents a navigation menu item with hierarchical support.

| Property | Type | Description |
|----------|------|-------------|
| `Id` | `int` | Unique identifier |
| `ParentId` | `int?` | Parent navigation item ID (null for root items) |
| `Name` | `string` | Internal name/identifier |
| `Title` | `string` | Display title |
| `Url` | `string?` | Navigation URL |
| `IconClass` | `string?` | CSS class for icon (e.g., FontAwesome) |
| `SortOrder` | `int` | Display order within parent |
| `IsActive` | `bool` | Whether the item is active |
| `IsExternalUrl` | `bool` | Whether URL opens in new window |
| `RequiresIframe` | `bool` | Whether content should load in iframe |
| `CreatedOn` | `DateTime` | Creation timestamp |
| `CreatedBy` | `string` | Creator identifier |
| `ModifiedOn` | `DateTime?` | Last modification timestamp |
| `ModifiedBy` | `string?` | Last modifier identifier |
| `Children` | `List<NavigationItem>` | Child navigation items |

### NavigationHistory

Represents a user's navigation history entry.

| Property | Type | Description |
|----------|------|-------------|
| `Id` | `int` | Unique identifier |
| `UserId` | `string` | User identifier |
| `Url` | `string` | Visited URL |
| `DisplayLabel` | `string?` | Human-readable label |
| `AccessedAt` | `DateTime` | Access timestamp |
| `NavigationType` | `int?` | Navigation type identifier |

## Error Handling

All service methods return `ApiResponse<T>` which provides detailed HTTP response information including status codes, headers, and error details. This allows for comprehensive error handling without relying solely on exceptions.

```csharp
var response = await _navigationItemService.GetAllAsync();

if (response.IsSuccessStatusCode)
{
    var items = response.Content;
    // Process items
}
else
{
    // Handle HTTP errors using status code
    switch (response.StatusCode)
    {
        case System.Net.HttpStatusCode.NotFound:
            _logger.LogWarning("Resource not found");
            break;
        case System.Net.HttpStatusCode.Unauthorized:
            _logger.LogWarning("Authentication failed");
            break;
        default:
            _logger.LogError("API error: {StatusCode}", response.StatusCode);
            break;
    }
}
```

For network-level errors, exceptions are still thrown:

```csharp
try
{
    var response = await _navigationItemService.GetAllAsync();
}
catch (HttpRequestException ex)
{
    // Handle network/HTTP errors
    _logger.LogError(ex, "Failed to connect to VoyagerUx API");
}
catch (Exception ex)
{
    // Handle other errors
    _logger.LogError(ex, "Unexpected error occurred");
}
```

### Common Exceptions

| Exception | Cause |
|-----------|-------|
| `ConfigurationErrorsException` | Missing or invalid configuration settings |
| `HttpRequestException` | Network connectivity or HTTP errors |
| `Refit.ApiException` | API returned an error response |

## Logging

The library uses `Corp.Lib.Logging` for all logging operations. Log entries include:

- **Debug**: Method entry/exit, return values
- **Information**: Successful write operations (Insert, Update, Delete)
- **Error**: Exception details with full context

Ensure your application has configured the logging infrastructure appropriately.

## Dependencies

| Package | Version | Description |
|---------|---------|-------------|
| `Corp.Lib.Cryptography` | 10.1.1 | AES encryption for secure password handling |
| `Corp.Lib.Refit` | 10.1.5 | Refit HTTP client configuration extensions |
| `Refit` | 8.0.0+ | Type-safe REST client with `ApiResponse<T>` support |
| `Corp.Api.VoyagerUx.Obj` | (Project Reference) | Entity definitions |

## Troubleshooting

### Configuration Errors

**Error:** `VoyagerUx API library configuration error. TargetedVoyagerInstance configuration element does not exist in appsettings.json.`

**Solution:** Ensure your `appsettings.json` contains the `TargetedVoyagerInstance` key with a valid value.

---

**Error:** `VoyagerUx API library configuration error. TargetedVoyagerEnvironment configuration element does not exist in appsettings.json.`

**Solution:** Ensure your `appsettings.json` contains the `TargetedVoyagerEnvironment` key with a valid value.

---

**Error:** `VoyagerUx API library configuration error. {Instance}.{Environment}.Corp.Api.VoyagerUx.Url configuration element does not exist or does not have a value.`

**Solution:** Add the required URL configuration key with the correct instance and environment prefix (e.g., `Mcs.Dev.Corp.Api.VoyagerUx.Url`).

---

**Error:** `VoyagerUx API library configuration error. Either ({prefix}.CertificatePath and {prefix}.Password) or {prefix}.CertificateThumbprint must be configured.`

**Solution:** Provide either a certificate path with an AES-encrypted password, or a certificate thumbprint from the machine store. See the [Configuration](#configuration) section for details.

---

### Certificate Errors

**Error:** `The remote certificate is invalid according to the validation procedure.`

**Solution:** 
1. Verify the certificate file exists at the configured path
2. Ensure the certificate has not expired
3. Verify the certificate password is correctly encrypted

---

### Connection Errors

**Error:** `HttpRequestException: Connection refused`

**Solution:**
1. Verify the API URL is correct and accessible
2. Check firewall rules
3. Verify VPN connection if required

---

## Version History

| Version | Date | Notes |
|---------|------|-------|
| 10.2.0 | 2025 | Added `DeleteByUserIdAsync` to `INavigationHistoryService`, added `NavigationType` to `NavigationHistory`, updated dependencies |
| 10.1.0 | 2025 | Added `ApiResponse<T>` return types, migrated to `HybridCache` |
| 10.0.0 | 2024 | Initial release targeting .NET 10 |

## Support

For issues or questions, contact the Sedgwick Consumer Claims development team.

---

*© Sedgwick Consumer Claims. All rights reserved.*
