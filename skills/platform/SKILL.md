---
name: platform
description: >
  Use this skill whenever platform-specific code is written in Consulo. Trigger on:
  "Platform", "PlatformFileSystem", "PlatformOperatingSystem", "PlatformUser", "PlatformJvm",
  "Platform.current()", "platform.os()", "platform.fs()", "platform.user()", "platform.jvm()",
  "isWindows", "isMac", "isLinux", "homePath", "getEnvironmentVariable", "environmentVariables",
  "getPath", "collectHomePaths", "BundleType", "System.getenv", "System.getProperty",
  "Path.of", "user.home", "CpuArchitecture", "LineSeparator", "ProcessInfo",
  or any request involving OS detection, file paths, environment variables, or user home dir.
  MUST be used proactively whenever any Platform API is used or platform-specific paths are built.
---

# Consulo Platform API

The `Platform` API (`consulo.platform`) abstracts OS-specific functionality across
Windows, macOS, and Linux. **Always use it** — never call `System.getenv()`,
`System.getProperty("user.home")`, or `Path.of()` for platform-dependent paths directly.

Module to require: `consulo.platform.api`

---

## Entry Point — `Platform.current()`

```java
Platform platform = Platform.current(); // gets the running platform
```

`Platform` gives access to four sub-interfaces:

| Sub-interface | Access | Purpose |
|---|---|---|
| `PlatformFileSystem` | `platform.fs()` | Build paths, check case-sensitivity |
| `PlatformOperatingSystem` | `platform.os()` | OS detection, env vars, processes |
| `PlatformUser` | `platform.user()` | Home dir, username, dark theme |
| `PlatformJvm` | `platform.jvm()` | JVM version, vendor, CPU arch |

---

## PlatformFileSystem — `platform.fs()`

**Always use this instead of `Path.of()`** for building file system paths.

```java
// ✅ Correct — platform-aware path construction
Path cli     = platform.fs().getPath("/usr/bin/arduino-cli");
Path subdir  = platform.fs().getPath("/base", "sub", "dir");       // varargs segments
Path appData = platform.fs().getPath(localAppData, "Arduino15", "arduino-cli.exe");

// ❌ Wrong — bypasses platform abstraction
Path cli = Path.of("/usr/bin/arduino-cli");
```

### Methods

```java
Path getPath(String path);
Path getPath(String path, String... more);   // segments joined with platform separator
boolean isCaseSensitive();                   // false on Windows/macOS HFS+, true on Linux
boolean areSymLinksSupported();
```

---

## PlatformOperatingSystem — `platform.os()`

### OS Detection

```java
if (platform.os().isWindows()) { ... }
if (platform.os().isMac())     { ... }
if (platform.os().isLinux())   { ... }
if (platform.os().isFreeBSD()) { ... }
if (platform.os().isUnix())    { ... }  // Linux + macOS + FreeBSD
```

### Environment Variables

**Always use this instead of `System.getenv()`.**

```java
// Returns @Nullable — always check for null
@Nullable String val = platform.os().getEnvironmentVariable("MY_VAR");

// Returns @Nullable even with default — check result or use ternary
@Nullable String val = platform.os().getEnvironmentVariable("MY_VAR", "default");

// Safe pattern — explicit null guard
String localAppData = platform.os().getEnvironmentVariable("LOCALAPPDATA");
if (localAppData != null) {
    tryPath(platform.fs().getPath(localAppData, "MyApp", "tool.exe"), consumer);
}

// Safe pattern — ternary with fallback
String pf = platform.os().getEnvironmentVariable("ProgramFiles");
Path toolPath = platform.fs().getPath(pf != null ? pf : "C:\\Program Files", "MyApp", "tool.exe");

// All variables as a map
Map<String, String> env = platform.os().environmentVariables();
```

> ⚠️ `getEnvironmentVariable(key, default)` is declared `@Nullable` even though it
> falls back to `defaultValue`. NullAway will complain if you pass the result directly
> to non-null parameters — always use a null check or ternary.

### OS Metadata

```java
String name    = platform.os().name();     // e.g. "Linux"
String version = platform.os().version();  // e.g. "5.15.0"
String arch    = platform.os().arch();     // e.g. "amd64"
LineSeparator sep = platform.os().lineSeparator(); // LF, CRLF, or CR
```

### Windows-Specific (cast to `WindowsOperatingSystem`)

```java
if (platform.os() instanceof WindowsOperatingSystem win) {
    boolean w11 = win.isWindows11OrNewer();
    boolean w10 = win.isWindows10OrNewer();
}
```

### Unix-Specific (cast to `UnixOperationSystem`)

```java
if (platform.os() instanceof UnixOperationSystem unix) {
    boolean wayland = unix.isWayland();
    boolean kde     = unix.isKDE();
}
```

---

## PlatformUser — `platform.user()`

**Always use this instead of `System.getProperty("user.home")`.**

```java
Path home = platform.user().homePath();       // home directory as Path

// ✅ Resolve sub-paths using standard Path API
Path localBin = platform.user().homePath().resolve(".local/bin/my-tool");
Path config   = platform.user().homePath().resolve(".config/myapp/settings.json");

// ❌ Wrong — bypasses platform abstraction
Path home = Path.of(System.getProperty("user.home"));
```

### Methods

```java
Path    homePath();    // user's home directory
String  name();        // login name
boolean superUser();   // true if running as root/admin
boolean darkTheme();   // OS dark mode preference
```

---

## PlatformJvm — `platform.jvm()`

```java
String   version        = platform.jvm().version();         // e.g. "21"
String   runtimeVersion = platform.jvm().runtimeVersion();
String   vendor         = platform.jvm().vendor();
String   name           = platform.jvm().name();
boolean  is64bit        = platform.jvm().isAny64Bit();

CpuArchitecture arch = platform.jvm().arch();
// Constants: CpuArchitecture.X86, X86_64, AARCH64, RISCV64, LOONG64, E2K

// JVM system properties
@Nullable String prop = platform.jvm().getRuntimeProperty("java.io.tmpdir");
```

---

## Platform Executable Name Mapping

`Platform` provides helpers for naming executables and libraries correctly per OS:

```java
// Maps "arduino-cli" → "arduino-cli" (Unix) or "arduino-cli.exe" (Windows)
String exe = platform.mapAnyExecutableName("arduino-cli");

// Maps "mylib" → "libmylib.so" (Linux) / "libmylib.dylib" (macOS) / "mylib.dll" (Windows)
String lib = platform.mapLibraryName("mylib");
```

---

## Canonical Pattern — `collectHomePaths` in BundleType

```java
@Override
public void collectHomePaths(Platform platform, Consumer<Path> consumer) {
    if (platform.os().isWindows()) {
        String localAppData = platform.os().getEnvironmentVariable("LOCALAPPDATA");
        if (localAppData != null) {
            tryPath(platform.fs().getPath(localAppData, "MyApp", "tool.exe"), consumer);
        }
        String pf = platform.os().getEnvironmentVariable("ProgramFiles");
        tryPath(platform.fs().getPath(pf != null ? pf : "C:\\Program Files", "MyApp", "tool.exe"), consumer);
    }
    else if (platform.os().isMac()) {
        tryPath(platform.fs().getPath("/usr/local/bin/my-tool"), consumer);
        tryPath(platform.fs().getPath("/opt/homebrew/bin/my-tool"), consumer);
    }
    else {
        tryPath(platform.fs().getPath("/usr/bin/my-tool"), consumer);
        tryPath(platform.fs().getPath("/usr/local/bin/my-tool"), consumer);
        tryPath(platform.user().homePath().resolve(".local/bin/my-tool"), consumer);
    }
}

private static void tryPath(Path path, Consumer<Path> consumer) {
    if (Files.isExecutable(path)) {
        consumer.accept(path);
    }
}
```

---

## What NOT to Do

```java
// ❌ Raw System calls — not platform-abstracted
System.getenv("LOCALAPPDATA")
System.getProperty("user.home")
Path.of(System.getProperty("user.home"), ".local", "bin", "tool")

// ❌ Using Path.of for platform-dependent paths
Path.of("/usr/bin/my-tool")     // use platform.fs().getPath() instead

// ❌ Passing @Nullable result of getEnvironmentVariable directly to non-null parameter
platform.fs().getPath(platform.os().getEnvironmentVariable("VAR", ""), "sub")  // NullAway error
```
