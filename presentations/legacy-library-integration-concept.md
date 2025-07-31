# Legacy .NET Library Integration Concept

## Overview

This document outlines a concept for integrating a legacy .NET library into a modern .NET Core Web API while maintaining backward compatibility and implementing modern practices such as dependency injection, structured logging, and telemetry.

## Problem Statement

Legacy .NET libraries often have the following characteristics that make them challenging to integrate into modern applications:

- **No Dependency Injection**: Object instances are created using `new` throughout the codebase
- **Legacy Logging**: Uses `Debug.WriteLine` instead of structured logging interfaces
- **No Telemetry**: Performance measurements are done via `Debug.WriteLine` instead of proper telemetry
- **Tight Coupling**: Direct dependencies make testing and mocking difficult

## Solution Architecture

The solution involves creating a modern wrapper around the legacy library that:

1. **Uses the original legacy implementation** - the wrapper delegates to the actual legacy code
2. **Enhances through property injection** - modern dependencies are added via properties
3. **Maintains constructor compatibility** - no changes to existing constructors
4. **Provides conditional modern features** - logging and telemetry work when available, fallback to Debug.WriteLine when not
5. **Enables simple registration** via `builder.Services.AddLegacyLibrary()`
6. **Allows gradual migration** of existing consumers

### Key Architectural Principle

**Augmentation over Reimplementation**: Instead of reimplementing the legacy logic, the wrapper:
- Creates an instance of the original `LegacyDataProcessor`
- Sets modern dependencies (ILogger, telemetry) as properties
- Delegates all method calls to the original implementation
- Adds distributed tracing around the legacy calls

This ensures that all existing business logic, edge cases, and behaviors are preserved exactly as they were.

## Implementation Example

### 1. Legacy Library Interface (Example)

First, let's define what a typical legacy library might look like:

```csharp
// Original legacy library interface
public interface ILegacyDataProcessor
{
    ProcessResult ProcessData(string data);
    void ProcessBatch(IEnumerable<string> items);
}

// Legacy implementation with optional modern properties
public class LegacyDataProcessor : ILegacyDataProcessor
{
    // Optional modern dependencies - can be set via properties
    public ILogger? Logger { get; set; }
    public ILoggerFactory? LoggerFactory { get; set; }
    public Counter<long>? ProcessCounter { get; set; }
    public Histogram<double>? ProcessingDuration { get; set; }
    
    // Default constructor can optionally initialize with debug logger factory
    public LegacyDataProcessor()
    {
        // Uncomment to enable structured logging by default in legacy scenarios
        // LoggerFactory = new DebugLoggerFactory();
        // Logger = LoggerFactory.CreateLogger<LegacyDataProcessor>();
    }
    
    public ProcessResult ProcessData(string data)
    {
        // Legacy approach - direct instantiation
        var validator = new DataValidator();
        var transformer = new DataTransformer();
        
        // Set modern properties on legacy components if available
        if (LoggerFactory != null)
        {
            validator.Logger = LoggerFactory.CreateLogger<DataValidator>();
            transformer.Logger = LoggerFactory.CreateLogger<DataTransformer>();
        }
        else if (Logger != null)
        {
            // Fallback to shared logger if factory is not available
            validator.Logger = Logger;
            transformer.Logger = Logger;
        }
        
        // Modern logging with fallback to legacy
        if (Logger != null)
            Logger.LogInformation("Processing data with length: {DataLength}", data?.Length ?? 0);
        else
            Debug.WriteLine($"Processing data: {data}");
        
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            // Processing logic (unchanged)
            var validationResult = validator.Validate(data);
            if (!validationResult.IsValid)
            {
                if (Logger != null)
                    Logger.LogWarning("Validation failed: {Error}", validationResult.Error);
                else
                    Debug.WriteLine($"Validation failed: {validationResult.Error}");
                    
                ProcessCounter?.Add(1, new KeyValuePair<string, object?>("result", "validation_failed"));
                return ProcessResult.Failed(validationResult.Error);
            }
            
            var result = transformer.Transform(data);
            
            stopwatch.Stop();
            
            // Modern telemetry with fallback to legacy logging
            ProcessingDuration?.Record(stopwatch.Elapsed.TotalMilliseconds);
            ProcessCounter?.Add(1, new KeyValuePair<string, object?>("result", "success"));
            
            if (Logger != null)
                Logger.LogInformation("Processing completed successfully in {Duration}ms", stopwatch.ElapsedMilliseconds);
            else
                Debug.WriteLine($"Processing completed in {stopwatch.ElapsedMilliseconds}ms");
            
            return ProcessResult.Success(result);
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            ProcessCounter?.Add(1, new KeyValuePair<string, object?>("result", "error"));
            
            if (Logger != null)
                Logger.LogError(ex, "Error processing data");
            else
                Debug.WriteLine($"Error processing data: {ex.Message}");
                
            return ProcessResult.Failed(ex.Message);
        }
    }
    
    public void ProcessBatch(IEnumerable<string> items)
    {
        var itemsList = items.ToList();
        
        if (Logger != null)
            Logger.LogInformation("Starting batch processing of {ItemCount} items", itemsList.Count);
        else
            Debug.WriteLine($"Starting batch processing of {itemsList.Count} items");
        
        var successCount = 0;
        var failureCount = 0;
        
        foreach (var item in itemsList)
        {
            var result = ProcessData(item);
            if (result.IsSuccess)
                successCount++;
            else
                failureCount++;
        }
        
        if (Logger != null)
            Logger.LogInformation("Batch processing completed. Success: {SuccessCount}, Failures: {FailureCount}", 
                successCount, failureCount);
        else
            Debug.WriteLine("Batch processing completed");
    }
}

public class ProcessResult
{
    public bool IsSuccess { get; set; }
    public string Data { get; set; }
    public string Error { get; set; }
    
    public static ProcessResult Success(string data) => new() { IsSuccess = true, Data = data };
    public static ProcessResult Failed(string error) => new() { IsSuccess = false, Error = error };
}
```

### 2. Modern Wrapper Implementation

Now, let's create a modern wrapper that implements dependency injection, logging, and telemetry:

```csharp
using Microsoft.Extensions.Logging;
using System.Diagnostics;
using System.Diagnostics.Metrics;

// Legacy internal classes with optional modern properties
public class DataValidator
{
    // Optional modern dependency - can be set via property
    public ILogger? Logger { get; set; }
    
    public DataValidator()
    {
        // Uncomment to enable structured logging by default in legacy scenarios
        // Logger = new DebugLoggerFactory().CreateLogger<DataValidator>();
    }
    
    public ValidationResult Validate(string data)
    {
        // Modern logging with fallback to legacy
        if (Logger != null)
            Logger.LogDebug("Validating data: {Data}", data);
        else
            Debug.WriteLine($"Validating data: {data}");
        
        // Validation logic here (unchanged)
        if (string.IsNullOrWhiteSpace(data))
        {
            if (Logger != null)
                Logger.LogWarning("Data validation failed: empty or null data");
            else
                Debug.WriteLine("Data validation failed: empty or null data");
                
            return ValidationResult.Invalid("Data cannot be empty");
        }
        
        if (Logger != null)
            Logger.LogDebug("Data validation successful");
        else
            Debug.WriteLine("Data validation successful");
            
        return ValidationResult.Valid();
    }
}

public class DataTransformer
{
    // Optional modern dependency - can be set via property
    public ILogger? Logger { get; set; }
    
    public DataTransformer()
    {
        // Uncomment to enable structured logging by default in legacy scenarios
        // Logger = new DebugLoggerFactory().CreateLogger<DataTransformer>();
    }
    
    public string Transform(string data)
    {
        // Modern logging with fallback to legacy
        if (Logger != null)
            Logger.LogDebug("Transforming data");
        else
            Debug.WriteLine("Transforming data");
        
        // Transformation logic here (unchanged)
        var result = data.ToUpperInvariant();
        
        if (Logger != null)
            Logger.LogDebug("Data transformation completed");
        else
            Debug.WriteLine("Data transformation completed");
            
        return result;
    }
}

// Modern wrapper that enhances the legacy implementation
public class ModernDataProcessorWrapper : ILegacyDataProcessor
{
    private readonly LegacyDataProcessor _legacyProcessor;
    private readonly ILoggerFactory _loggerFactory;
    private readonly Counter<long> _processCounter;
    private readonly Histogram<double> _processingDuration;
    
    public ModernDataProcessorWrapper(
        ILoggerFactory loggerFactory,
        IMeterFactory meterFactory)
    {
        // Create the legacy processor and enhance it with modern dependencies
        _legacyProcessor = new LegacyDataProcessor();
        _loggerFactory = loggerFactory;
        
        // Initialize telemetry
        var meter = meterFactory.Create("LegacyLibrary.DataProcessor");
        _processCounter = meter.CreateCounter<long>("data_processing_total", description: "Total number of data processing operations");
        _processingDuration = meter.CreateHistogram<double>("data_processing_duration", unit: "ms", description: "Duration of data processing operations");
        
        // Inject modern dependencies into the legacy processor
        _legacyProcessor.Logger = _loggerFactory.CreateLogger<LegacyDataProcessor>();
        _legacyProcessor.LoggerFactory = _loggerFactory;
        _legacyProcessor.ProcessCounter = _processCounter;
        _legacyProcessor.ProcessingDuration = _processingDuration;
    }
    
    public ProcessResult ProcessData(string data)
    {
        // Add distributed tracing around the legacy implementation
        using var activity = Activity.Current?.Source.StartActivity("ProcessData");
        activity?.SetTag("data.length", data?.Length ?? 0);
        
        // Delegate to the actual legacy implementation
        var result = _legacyProcessor.ProcessData(data);
        
        // Add additional telemetry tags based on result
        activity?.SetTag("result", result.IsSuccess ? "success" : "failed");
        
        return result;
    }
    
    public void ProcessBatch(IEnumerable<string> items)
    {
        // Add distributed tracing around the legacy implementation
        using var activity = Activity.Current?.Source.StartActivity("ProcessBatch");
        var itemsList = items.ToList();
        activity?.SetTag("batch.size", itemsList.Count);
        
        // Delegate to the actual legacy implementation
        _legacyProcessor.ProcessBatch(itemsList);
    }
}

public class ValidationResult
{
    public bool IsValid { get; set; }
    public string Error { get; set; }
    
    public static ValidationResult Valid() => new() { IsValid = true };
    public static ValidationResult Invalid(string error) => new() { IsValid = false, Error = error };
}
```

### 3. Debug LoggerFactory for Legacy Scenarios

For scenarios where no DI container is available but you still want to use structured logging instead of `Debug.WriteLine`, you can use this custom `LoggerFactory`:

```csharp
using Microsoft.Extensions.Logging;
using System.Diagnostics;

/// <summary>
/// A LoggerFactory that creates loggers which output to System.Diagnostics.Debug.WriteLine
/// with proper log level formatting and exception handling.
/// Useful for legacy scenarios where no DI container is available.
/// </summary>
public class DebugLoggerFactory : ILoggerFactory
{
    private readonly LogLevel _minimumLevel;
    
    public DebugLoggerFactory(LogLevel minimumLevel = LogLevel.Debug)
    {
        _minimumLevel = minimumLevel;
    }
    
    public ILogger CreateLogger(string categoryName)
    {
        return new DebugLogger(categoryName, _minimumLevel);
    }
    
    public void AddProvider(ILoggerProvider provider)
    {
        // Not supported in this simple implementation
    }
    
    public void Dispose()
    {
        // Nothing to dispose
    }
}

/// <summary>
/// A logger implementation that outputs to System.Diagnostics.Debug.WriteLine
/// with formatted log levels and proper exception handling.
/// </summary>
public class DebugLogger : ILogger
{
    private readonly string _categoryName;
    private readonly LogLevel _minimumLevel;
    
    public DebugLogger(string categoryName, LogLevel minimumLevel)
    {
        _categoryName = categoryName;
        _minimumLevel = minimumLevel;
    }
    
    public IDisposable BeginScope<TState>(TState state) => new NoOpDisposable();
    
    public bool IsEnabled(LogLevel logLevel) => logLevel >= _minimumLevel;
    
    public void Log<TState>(LogLevel logLevel, EventId eventId, TState state, Exception exception, Func<TState, Exception, string> formatter)
    {
        if (!IsEnabled(logLevel))
            return;
            
        var levelLabel = GetLogLevelLabel(logLevel);
        var message = formatter(state, exception);
        var timestamp = DateTime.UtcNow.ToString("yyyy-MM-dd HH:mm:ss.fff");
        
        var formattedMessage = $"{timestamp} [{levelLabel}] {_categoryName}: {message}";
        
        // Include exception details if present
        if (exception != null)
        {
            formattedMessage += Environment.NewLine + 
                               $"Exception: {exception.GetType().Name}: {exception.Message}" +
                               Environment.NewLine + 
                               $"StackTrace: {exception.StackTrace}";
        }
        
        Debug.WriteLine(formattedMessage);
    }
    
    private static string GetLogLevelLabel(LogLevel logLevel)
    {
        return logLevel switch
        {
            LogLevel.Trace => "TRACE",
            LogLevel.Debug => "DEBUG", 
            LogLevel.Information => "INFO",
            LogLevel.Warning => "WARN",
            LogLevel.Error => "ERROR",
            LogLevel.Critical => "CRIT",
            _ => "NONE"
        };
    }
    
    private class NoOpDisposable : IDisposable
    {
        public void Dispose() { }
    }
}

/// <summary>
/// Usage example for legacy scenarios without DI container
/// </summary>
public static class LegacyUsageExample
{
    public static void DemonstrateUsage()
    {
        // Create a debug logger factory for legacy scenarios
        var loggerFactory = new DebugLoggerFactory(LogLevel.Debug);
        
        // Create and configure a legacy processor with the debug logger factory
        var processor = new LegacyDataProcessor();
        processor.LoggerFactory = loggerFactory;
        
        // Now all logging will go through structured logging to Debug.WriteLine
        // with proper formatting: "[INFO] LegacyDataProcessor: Processing data with length: 5"
        var result = processor.ProcessData("test");
        
        // The logger factory can also be used for individual components
        var validator = new DataValidator();
        validator.Logger = loggerFactory.CreateLogger<DataValidator>();
        
        var transformer = new DataTransformer(); 
        transformer.Logger = loggerFactory.CreateLogger<DataTransformer>();
    }
}
```

### 4. Service Registration Extension

Create an extension method for easy service registration:

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection.Extensions;

public static class LegacyLibraryServiceCollectionExtensions
{
    /// <summary>
    /// Adds the legacy library services to the DI container with modern patterns
    /// </summary>
    /// <param name="services">The service collection</param>
    /// <param name="configureOptions">Optional configuration for the legacy library</param>
    /// <returns>The service collection for chaining</returns>
    public static IServiceCollection AddLegacyLibrary(
        this IServiceCollection services, 
        Action<LegacyLibraryOptions>? configureOptions = null)
    {
        // Register configuration if provided
        if (configureOptions != null)
        {
            services.Configure(configureOptions);
        }
        
        // Register the modern wrapper that uses the legacy implementation internally
        services.TryAddScoped<ILegacyDataProcessor, ModernDataProcessorWrapper>();
        
        return services;
    }
    
    /// <summary>
    /// Adds the legacy library services with only the original implementation
    /// for consumers that want to maintain the exact legacy behavior
    /// </summary>
    public static IServiceCollection AddLegacyLibraryClassic(this IServiceCollection services)
    {
        services.TryAddScoped<ILegacyDataProcessor, LegacyDataProcessor>();
        return services;
    }
}

public class LegacyLibraryOptions
{
    public bool EnableDetailedLogging { get; set; } = true;
    public bool EnableTelemetry { get; set; } = true;
    public TimeSpan ProcessingTimeout { get; set; } = TimeSpan.FromMinutes(5);
}
```

### 4. Usage in Modern .NET Core API

Here's how to use the integrated library in a modern .NET Core application:

```csharp
// Program.cs
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Add the legacy library with modern patterns
builder.Services.AddLegacyLibrary(options =>
{
    options.EnableDetailedLogging = true;
    options.EnableTelemetry = true;
    options.ProcessingTimeout = TimeSpan.FromMinutes(10);
});

// Add telemetry
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics =>
    {
        metrics.AddMeter("LegacyLibrary.DataProcessor");
    });

var app = builder.Build();

// Configure the HTTP request pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();

// Controller using the legacy library
[ApiController]
[Route("api/[controller]")]
public class DataController : ControllerBase
{
    private readonly ILegacyDataProcessor _dataProcessor;
    private readonly ILogger<DataController> _logger;
    
    public DataController(ILegacyDataProcessor dataProcessor, ILogger<DataController> logger)
    {
        _dataProcessor = dataProcessor;
        _logger = logger;
    }
    
    [HttpPost("process")]
    public async Task<IActionResult> ProcessData([FromBody] ProcessDataRequest request)
    {
        _logger.LogInformation("Received data processing request");
        
        try
        {
            var result = _dataProcessor.ProcessData(request.Data);
            
            if (result.IsSuccess)
            {
                return Ok(new { Success = true, Data = result.Data });
            }
            else
            {
                return BadRequest(new { Success = false, Error = result.Error });
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unexpected error processing data");
            return StatusCode(500, new { Success = false, Error = "Internal server error" });
        }
    }
    
    [HttpPost("process-batch")]
    public async Task<IActionResult> ProcessBatch([FromBody] ProcessBatchRequest request)
    {
        _logger.LogInformation("Received batch processing request with {Count} items", request.Items.Count);
        
        try
        {
            // Run batch processing in background to avoid blocking the request
            _ = Task.Run(() => _dataProcessor.ProcessBatch(request.Items));
            
            return Accepted(new { Message = "Batch processing started" });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error starting batch processing");
            return StatusCode(500, new { Success = false, Error = "Internal server error" });
        }
    }
}

public record ProcessDataRequest(string Data);
public record ProcessBatchRequest(List<string> Items);
```

### 5. Migration Strategy for Existing Consumers

For applications already using the legacy library, migration can be gradual:

```csharp
// Existing legacy consumer (no changes required)
public class ExistingLegacyService
{
    public void DoSomething()
    {
        // This continues to work exactly as before
        var processor = new LegacyDataProcessor();
        var result = processor.ProcessData("some data");
    }
}

// Modernized consumer using DI
public class ModernizedService
{
    private readonly ILegacyDataProcessor _processor;
    
    public ModernizedService(ILegacyDataProcessor processor)
    {
        _processor = processor;
    }
    
    public void DoSomething()
    {
        // Uses the modern wrapper with DI, logging, and telemetry
        var result = _processor.ProcessData("some data");
    }
}

// Hybrid approach - gradually modernize specific parts
public class HybridService
{
    private readonly ILegacyDataProcessor _modernProcessor;
    private readonly LegacyDataProcessor _legacyProcessor;
    
    public HybridService(ILegacyDataProcessor modernProcessor)
    {
        _modernProcessor = modernProcessor;
        _legacyProcessor = new LegacyDataProcessor(); // Fall back when needed
    }
    
    public void ProcessCriticalData(string data)
    {
        // Use modern implementation for critical operations
        var result = _modernProcessor.ProcessData(data);
    }
    
    public void ProcessLegacyData(string data)
    {
        // Use legacy implementation for specific cases
        var result = _legacyProcessor.ProcessData(data);
    }
}
```

### 6. Using DebugLoggerFactory for Enhanced Legacy Scenarios

To replace all `Debug.WriteLine` calls with structured logging even in pure legacy scenarios, you can enable the `DebugLoggerFactory` by default:

```csharp
// Option 1: Enable in constructors (uncomment the lines in legacy classes)
public class LegacyDataProcessor : ILegacyDataProcessor
{
    public ILogger? Logger { get; set; }
    public ILoggerFactory? LoggerFactory { get; set; }
    
    public LegacyDataProcessor()
    {
        // Enable structured logging by default in legacy scenarios
        LoggerFactory = new DebugLoggerFactory();
        Logger = LoggerFactory.CreateLogger<LegacyDataProcessor>();
    }
    // ... rest of implementation
}

// Option 2: Factory method for creating enhanced legacy instances
public static class LegacyDataProcessorFactory
{
    public static LegacyDataProcessor CreateWithStructuredLogging()
    {
        var processor = new LegacyDataProcessor();
        var loggerFactory = new DebugLoggerFactory();
        processor.LoggerFactory = loggerFactory;
        processor.Logger = loggerFactory.CreateLogger<LegacyDataProcessor>();
        return processor;
    }
    
    public static LegacyDataProcessor CreateClassic()
    {
        return new LegacyDataProcessor(); // No structured logging
    }
}

// Option 3: Global configuration for legacy usage
public static class LegacyLibraryGlobalConfig
{
    public static bool EnableStructuredLoggingByDefault { get; set; } = false;
    
    public static ILoggerFactory? DefaultLoggerFactory { get; set; }
    
    public static void EnableDebugLogging(LogLevel minimumLevel = LogLevel.Debug)
    {
        DefaultLoggerFactory = new DebugLoggerFactory(minimumLevel);
        EnableStructuredLoggingByDefault = true;
    }
}

// Usage examples
public class LegacyUsageExamples
{
    public void ExampleWithFactoryMethod()
    {
        // Creates a processor with structured logging enabled
        var processor = LegacyDataProcessorFactory.CreateWithStructuredLogging();
        
        // All logging will now output to Debug.WriteLine with format:
        // "2023-12-07 10:30:45.123 [INFO] LegacyDataProcessor: Processing data with length: 5"
        var result = processor.ProcessData("hello");
    }
    
    public void ExampleWithGlobalConfig()
    {
        // Enable structured logging globally for all legacy instances
        LegacyLibraryGlobalConfig.EnableDebugLogging(LogLevel.Information);
        
        // Now all new instances will use structured logging by default
        var processor = new LegacyDataProcessor();
        var validator = new DataValidator();
        var transformer = new DataTransformer();
        
        // All will log to Debug.WriteLine with proper formatting
    }
    
    public void ExampleManualSetup()
    {
        var loggerFactory = new DebugLoggerFactory();
        
        var processor = new LegacyDataProcessor();
        processor.LoggerFactory = loggerFactory;
        processor.Logger = loggerFactory.CreateLogger<LegacyDataProcessor>();
        
        // Individual components will get their own logger categories
        var result = processor.ProcessData("test"); // Uses component-specific loggers internally
    }
}
```

This approach provides several benefits:

1. **Gradual Migration**: Legacy code can be enhanced with structured logging without requiring DI
2. **Better Debugging**: Log output includes timestamps, log levels, and proper exception formatting
3. **Component Categorization**: Different components get their own logger categories for better filtering
4. **Exception Handling**: Full exception details including stack traces are automatically included
5. **Backward Compatibility**: Can be enabled/disabled without breaking existing code

Sample output from `DebugLoggerFactory`:
```
2023-12-07 10:30:45.123 [INFO] LegacyDataProcessor: Processing data with length: 5
2023-12-07 10:30:45.124 [DEBUG] DataValidator: Validating data: hello
2023-12-07 10:30:45.125 [DEBUG] DataValidator: Data validation successful
2023-12-07 10:30:45.126 [DEBUG] DataTransformer: Transforming data
2023-12-07 10:30:45.127 [INFO] LegacyDataProcessor: Processing completed successfully in 4ms
```

## Benefits of This Approach

### 1. **True Legacy Code Reuse**
- The wrapper actually uses the original legacy implementation, not a reimplementation
- All existing logic, business rules, and edge cases are preserved
- No risk of introducing bugs through code reimplementation
- Legacy code remains the single source of truth

### 2. **Property-Based Enhancement**
- Modern dependencies are added through properties, not constructor changes
- Legacy classes can work with or without modern dependencies
- No breaking changes to existing constructors or method signatures
- Gradual modernization without touching core legacy logic

### 3. **Backward Compatibility**
- Existing consumers continue to work without any changes
- Original interface remains intact
- Legacy implementation is still available if needed
- Zero migration required for existing code

### 4. **Modern Patterns Through Augmentation**
- Dependency injection enables better testability and loose coupling
- Structured logging replaces Debug.WriteLine when logger is available
- Telemetry and metrics are added without changing core logic
- Activity tracing supports distributed tracing scenarios

### 5. **Gradual Migration**
- Consumers can migrate at their own pace
- Hybrid approaches allow selective modernization
- Risk is minimized through incremental adoption
- Legacy and modern approaches can coexist

### 6. **Enhanced Observability**
- Conditionally replace `Debug.WriteLine` with structured logging
- Add performance metrics and counters
- Enable distributed tracing
- Support for modern APM tools
- Fallback to original behavior when modern dependencies are not available

### 7. **Minimal Code Changes**
- Legacy classes only gain new optional properties
- Core business logic remains untouched
- Wrapper provides thin layer for modern integrations
- Original Debug.WriteLine calls are preserved as fallback

## Configuration and Customization

The approach supports various configuration options:

```csharp
// Minimal setup
builder.Services.AddLegacyLibrary();

// With configuration
builder.Services.AddLegacyLibrary(options =>
{
    options.EnableDetailedLogging = false; // Reduce log verbosity
    options.EnableTelemetry = true;        // Enable metrics collection
    options.ProcessingTimeout = TimeSpan.FromSeconds(30);
});

// For performance-critical scenarios, use the classic implementation
builder.Services.AddLegacyLibraryClassic();

// Custom implementations can be registered
builder.Services.AddScoped<IDataValidator, CustomDataValidator>();
builder.Services.AddScoped<IDataTransformer, OptimizedDataTransformer>();
```

## Testing Strategy

The modernized approach enables comprehensive testing:

```csharp
public class ModernDataProcessorWrapperTests
{
    [Test]
    public void ProcessData_ValidInput_ReturnsSuccess()
    {
        // Arrange
        var mockLogger = new Mock<ILogger<LegacyDataProcessor>>();
        var mockMeterFactory = new Mock<IMeterFactory>();
        var mockMeter = new Mock<Meter>("test");
        var mockCounter = new Mock<Counter<long>>();
        var mockHistogram = new Mock<Histogram<double>>();
        
        mockMeterFactory.Setup(f => f.Create("LegacyLibrary.DataProcessor")).Returns(mockMeter.Object);
        mockMeter.Setup(m => m.CreateCounter<long>(It.IsAny<string>(), It.IsAny<string>(), It.IsAny<string>()))
               .Returns(mockCounter.Object);
        mockMeter.Setup(m => m.CreateHistogram<double>(It.IsAny<string>(), It.IsAny<string>(), It.IsAny<string>()))
               .Returns(mockHistogram.Object);
        
        var wrapper = new ModernDataProcessorWrapper(mockLogger.Object, mockMeterFactory.Object);
        
        // Act
        var result = wrapper.ProcessData("test");
        
        // Assert
        Assert.That(result.IsSuccess, Is.True);
        Assert.That(result.Data, Is.EqualTo("TEST"));
        
        // Verify that the wrapper actually uses the legacy implementation
        // The legacy processor should have logged the processing
        mockLogger.Verify(
            x => x.Log(
                LogLevel.Information,
                It.IsAny<EventId>(),
                It.Is<It.IsAnyType>((o, t) => o.ToString().Contains("Processing data")),
                It.IsAny<Exception>(),
                It.IsAny<Func<It.IsAnyType, Exception, string>>()),
            Times.AtLeastOnce);
    }
    
    [Test]
    public void LegacyProcessor_WithoutModernDependencies_StillWorks()
    {
        // Arrange - use legacy processor directly without any modern features
        var legacyProcessor = new LegacyDataProcessor();
        
        // Act
        var result = legacyProcessor.ProcessData("test");
        
        // Assert - should work exactly as before
        Assert.That(result.IsSuccess, Is.True);
        Assert.That(result.Data, Is.EqualTo("TEST"));
        
        // No exceptions should be thrown, and Debug.WriteLine should have been used
    }
}
```

## Conclusion

This concept demonstrates how to integrate a legacy .NET library into a modern .NET Core application while:

- **Actually using the original legacy code** instead of reimplementing it
- **Enhancing through properties** rather than breaking constructor signatures
- Maintaining complete backward compatibility
- Implementing modern patterns (DI, structured logging, telemetry)
- Providing simple integration via `builder.Services.AddLegacyLibrary()`
- Enabling gradual migration for existing consumers
- Improving testability and observability

The key architectural principle is **augmentation over reimplementation**: the wrapper pattern allows the legacy library to benefit from modern .NET Core features by setting optional properties on the original classes, while preserving all existing business logic, edge cases, and behaviors. This makes it a practical solution for enterprise environments where legacy systems need to coexist with modern applications without the risk of introducing bugs through code duplication.