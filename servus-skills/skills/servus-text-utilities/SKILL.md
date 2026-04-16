---
name: servus-text-utilities
description: String case conversion and text parsing with Servus.Core — ToSnakeCase, ToKebabCase, ToDotCase, GetLines, LineKeyValueParser
invocable: true
---

# Text Utilities with Servus.Core

Use when converting string cases or parsing line-based key-value text data.

## When to Use

- Converting PascalCase/camelCase identifiers to snake_case, kebab-case, or dot.case
- Parsing configuration or output text into key-value pairs
- Iterating over lines in a string lazily

## API Reference

### Case Conversion

**Namespace:** `Servus.Core.Text`

```csharp
public static class StringExtensions
{
    string ToSnakeCase(this string value);    // "MyClassName" → "my_class_name"
    string ToDotCase(this string value);      // "MyClassName" → "my.class.name"
    string ToKebabCase(this string value);    // "MyClassName" → "my-class-name"
}
```

Handles edge cases:
- Consecutive uppercase: `"IOStream"` → `"io_stream"`
- Digit boundaries: `"Port8080Open"` → `"port_8080_open"`
- Existing spaces: `"My Class"` → `"my_class"`

### Line Parsing

**Namespace:** `Servus.Core.Parsing`

```csharp
public static class StringExtensions
{
    // Lazy line-by-line iteration (handles \r\n, \r, \n)
    IEnumerable<string> GetLines(this string? value);
}
```

### Key-Value Parsing

**Namespace:** `Servus.Core.Parsing.Text`

```csharp
public class LineKeyValueParser
{
    // Split by single character
    LineKeyValueParser(char splitChar);

    // Split by regex pattern
    LineKeyValueParser(string splitRegex);

    // Parse multi-line text into key-value pairs
    IEnumerable<KeyValuePair<string, string>> Parse(string value);

    // Parse a single line
    KeyValuePair<string, string> ParseLine(string value);
}
```

## Usage Pattern

### Case conversion

```csharp
using Servus.Core.Text;

var typeName = typeof(OrderProcessingService).Name;

var snakeCase = typeName.ToSnakeCase();   // "order_processing_service"
var kebabCase = typeName.ToKebabCase();   // "order-processing-service"
var dotCase = typeName.ToDotCase();       // "order.processing.service"
```

### Parse config-style text

```csharp
using Servus.Core.Parsing.Text;

var parser = new LineKeyValueParser(':');

var text = """
    Host: localhost
    Port: 8080
    Protocol: https
    """;

foreach (var kvp in parser.Parse(text))
{
    Console.WriteLine($"{kvp.Key.Trim()} = {kvp.Value.Trim()}");
}
// Host = localhost
// Port = 8080
// Protocol = https
```

### Parse with regex delimiter

```csharp
// Split on "=" with optional whitespace
var parser = new LineKeyValueParser(@"\s*=\s*");

var line = "connectionString = Server=localhost;Database=mydb";
var kvp = parser.ParseLine(line);
// Key: "connectionString", Value: "Server=localhost;Database=mydb"
```

### Lazy line iteration

```csharp
using Servus.Core.Parsing;

string? logOutput = GetProcessOutput();

foreach (var line in logOutput.GetLines())
{
    if (line.Contains("ERROR"))
        ReportError(line);
}
// Handles null safely — returns empty enumerable
```

## Common Mistakes

- **Namespace collision on `StringExtensions`**: Both `Servus.Core.Text` and `Servus.Core.Parsing` define a `StringExtensions` class. Use explicit `using` to avoid ambiguity.
- **Expecting `ParseLine` to handle multi-line input**: `ParseLine` throws if the input contains line breaks. Use `Parse` for multi-line text.
- **Assuming `GetLines` splits on `\n` only**: It correctly handles `\r\n`, `\r`, and `\n` via `StringReader.ReadLine()`.
