---
name: process-spawn
description: >
  Use this skill whenever external processes are spawned in Consulo. Trigger on:
  "ProcessHandler", "ProcessHandlerBuilder", "ProcessHandlerBuilderFactory",
  "GeneralCommandLine", "ProcessListener", "ProcessEvent", "ProcessOutputType",
  "ProcessAdapter", "ProcessOutput", "CapturingProcessUtil", "ExecutionException",
  "startNotify", "waitFor", "destroyProcess", "onTextAvailable", "killable",
  "withExecutablePath", "withWorkingDirectory", "withPlatform", "withParameters",
  "withRedirectErrorStream", "addProcessListener", "getExitCode",
  "ProcessBuilder", "Runtime.exec", "ProcessBuilder.start",
  or any request to run an external command, tool, or subprocess.
  MUST be used proactively whenever any external process is launched.
---

# Consulo Process Spawn API

**Never use raw `ProcessBuilder` or `Runtime.exec()`** in Consulo plugin code.
Use `GeneralCommandLine` + `ProcessHandlerBuilderFactory` for cross-platform,
IDE-integrated process execution.

Modules to require: `consulo.process.api`

Key imports:
- `consulo.process.cmd.GeneralCommandLine`
- `consulo.process.ProcessHandlerBuilderFactory`
- `consulo.process.ProcessHandler`
- `consulo.process.ExecutionException`
- `consulo.process.event.ProcessListener`
- `consulo.process.event.ProcessEvent`
- `consulo.process.ProcessOutputType`
- `consulo.util.dataholder.Key`

---

## `GeneralCommandLine` — Building Commands

`GeneralCommandLine` is the OS-independent command builder. It handles quoting,
environment setup, and platform differences automatically.

### Construction — Fluent API

```java
Platform platform = Platform.current();

GeneralCommandLine commandLine = new GeneralCommandLine()
    .withExecutablePath(platform.fs().getPath("arduino-cli"))    // name only → found via PATH
    .withWorkingDirectory(sketchDir.toPath())
    .withPlatform(platform)                                       // always set this
    .withParameters("compile", "--fqbn", "arduino:avr:uno")
    .withRedirectErrorStream(true);                               // merge stderr into stdout
```

### Always Call `withPlatform(platform)`

Pass the `Platform` instance so the command line can apply platform-specific
escaping, PATH lookup, and environment handling:

```java
// ✅ Correct — platform context set
new GeneralCommandLine()
    .withExecutablePath(platform.fs().getPath("my-tool"))
    .withPlatform(platform);

// ❌ Wrong — no platform context, defaults to Platform.current() but loses
//            the caller's platform reference in BundleType callbacks
new GeneralCommandLine()
    .withExecutablePath(Path.of("my-tool"));
```

### Executable Path

Use `platform.fs().getPath()` — never `Path.of()` directly:

```java
// Absolute path (known location)
.withExecutablePath(platform.fs().getPath("/usr/bin/arduino-cli"))

// Name only — resolved via PATH at runtime
.withExecutablePath(platform.fs().getPath("arduino-cli"))

// Path passed in (e.g. from BundleType.getVersionString)
.withExecutablePath(path)   // path is already a java.nio.file.Path
```

### Adding Parameters

```java
// Fluent — for initial fixed args
new GeneralCommandLine().withParameters("compile", "--fqbn", board);

// Void — for conditional args after construction
commandLine.addParameter("compile");
if (boardFqbn != null && !boardFqbn.isBlank()) {
    commandLine.addParameters("--fqbn", boardFqbn);
}
commandLine.addParameter(sketchDir.getAbsolutePath());
```

### Environment Variables

```java
// Add single var (inherits parent env by default)
commandLine.withEnvironment("MY_VAR", "value");

// Control parent env inheritance
commandLine.withParentEnvironmentType(GeneralCommandLine.ParentEnvironmentType.CONSOLE); // default
commandLine.withParentEnvironmentType(GeneralCommandLine.ParentEnvironmentType.SYSTEM);
commandLine.withParentEnvironmentType(GeneralCommandLine.ParentEnvironmentType.NONE);    // isolated
```

---

## `ProcessHandlerBuilderFactory` — Creating Handlers

`ProcessHandlerBuilderFactory` is an application-level service. **Inject it via
constructor** — never call `Application.get().getInstance()` directly unless
injection is unavailable.

### Constructor Injection (preferred)

```java
@ExtensionImpl
public class MyManager {
    private final ProcessHandlerBuilderFactory myProcessHandlerBuilderFactory;

    @Inject
    public MyManager(ProcessHandlerBuilderFactory processHandlerBuilderFactory) {
        myProcessHandlerBuilderFactory = processHandlerBuilderFactory;
    }
}
```

### Passing to Non-Injected Classes

Task managers and similar classes are created by factories (not injected).
Receive the factory from the injected parent and pass it via constructor:

```java
// In injected manager:
@Override
public Supplier<? extends ExternalSystemTaskManager<MySettings>> getTaskManagerFactory() {
    return () -> new MyTaskManager(myProcessHandlerBuilderFactory);
}

// In non-injected task manager:
public class MyTaskManager {
    private final ProcessHandlerBuilderFactory myProcessHandlerBuilderFactory;

    public MyTaskManager(ProcessHandlerBuilderFactory processHandlerBuilderFactory) {
        myProcessHandlerBuilderFactory = processHandlerBuilderFactory;
    }
}
```

---

## Building and Running — Full Pattern

```java
private volatile @Nullable ProcessHandler myProcessHandler;

private void runCommand(GeneralCommandLine commandLine) throws ExternalSystemException {
    try {
        ProcessHandler handler = myProcessHandlerBuilderFactory
            .newBuilder(commandLine)
            .killable()       // supports destroyProcess() gracefully on Unix
            .build();

        myProcessHandler = handler;

        handler.addProcessListener(new ProcessListener() {
            @Override
            public void onTextAvailable(ProcessEvent event, Key outputType) {
                // stream output to wherever you need it
                String text = event.getText();
                boolean isStdout = ProcessOutputType.isStdout(outputType);
                // ...
            }

            @Override
            public void processTerminated(ProcessEvent event) {
                // called when process exits
            }
        });

        handler.startNotify();   // MUST call before waitFor()
        handler.waitFor();       // blocks until termination

        Integer exitCode = handler.getExitCode();
        if (exitCode != null && exitCode != 0) {
            throw new ExternalSystemException("Tool failed with exit code " + exitCode);
        }
    }
    catch (ExecutionException e) {
        throw new ExternalSystemException("Failed to launch: " + e.getMessage());
    }
    finally {
        myProcessHandler = null;
    }
}
```

### Cancellation Support

```java
// Cancel from another thread (e.g. cancelTask())
public boolean cancelTask(...) {
    ProcessHandler handler = myProcessHandler;
    if (handler != null) {
        handler.destroyProcess();
        return true;
    }
    return false;
}
```

---

## `ProcessHandlerBuilder` Options

| Method | Effect |
|---|---|
| `.killable()` | Creates `KillableProcessHandler` — supports graceful shutdown on Unix |
| `.colored()` | Enables ANSI colour escape decoding |
| `.silentReader()` | Low-priority I/O — for mostly silent processes |
| `.blockingReader()` | Blocking I/O — for high-throughput output |
| `.shouldKillProcessSoftly(bool)` | `true` = SIGTERM first, then SIGKILL; `false` = SIGKILL immediately |
| `.shouldDestroyProcessRecursively(bool)` | Kill child processes too |
| `.consoleType(ProcessConsoleType)` | `BUILTIN`, `EXTERNAL_EMULATION`, `EXTERNAL` (Windows only) |
| `.binary()` | Raw binary output — ignores most other options |

---

## `ProcessListener` — Receiving Output

`ProcessListener` is the interface to implement. `ProcessAdapter` is **deprecated** —
use `ProcessListener` directly (all methods have default no-op implementations):

```java
handler.addProcessListener(new ProcessListener() {
    @Override
    public void onTextAvailable(ProcessEvent event, Key outputType) {
        String text = event.getText();

        if (ProcessOutputType.isStdout(outputType)) {
            // stdout text
        }
        else if (ProcessOutputType.isStderr(outputType)) {
            // stderr text
        }
        // ProcessOutputType.SYSTEM is internal IDE messages — usually ignored
    }

    @Override
    public void processTerminated(ProcessEvent event) {
        int code = event.getExitCode();
    }

    @Override
    public void startNotified(ProcessEvent event) {
        // process has started
    }
});
```

### Output Type Checking

```java
// ✅ Correct — handles colour subtypes correctly
ProcessOutputType.isStdout(outputType)
ProcessOutputType.isStderr(outputType)

// ❌ Wrong — misses coloured subtypes (e.g. ANSI coloured stdout)
outputType == ProcessOutputType.STDOUT
ProcessOutputTypes.STDOUT.equals(outputType)
```

`ProcessOutputType` extends `Key<Object>`. The `Key` type parameter is
`consulo.util.dataholder.Key` — not `consulo.util.lang.Key`.

---

## Synchronous Output Capture — `CapturingProcessUtil`

For simple synchronous version checks or one-shot commands, use `CapturingProcessUtil`
instead of the verbose manual listener/StringBuilder pattern. No `ProcessHandlerBuilderFactory`
injection is needed — `CapturingProcessUtil` is a standalone utility.

Key imports:
- `consulo.process.util.CapturingProcessUtil`
- `consulo.process.util.ProcessOutput`

```java
@Override
public @Nullable String getVersionString(Platform platform, Path path) {
    try {
        GeneralCommandLine commandLine = new GeneralCommandLine()
            .withExecutablePath(path)
            .withPlatform(platform)
            .withParameters("version");

        ProcessOutput output = CapturingProcessUtil.execAndGetOutput(commandLine, 5000);

        Matcher matcher = VERSION_PATTERN.matcher(output.getStdout());
        if (matcher.find()) {
            return matcher.group(1);
        }
    }
    catch (ExecutionException ignored) {
    }
    return null;
}
```

### `ProcessOutput` API

```java
ProcessOutput output = CapturingProcessUtil.execAndGetOutput(commandLine, 5000);

output.getStdout()           // captured stdout as a single String
output.getStderr()           // captured stderr as a single String
output.getStdoutLines()      // stdout split into List<String>
output.getExitCode()         // process exit code (int)
output.isTimeout()           // true if the timeout elapsed before the process finished
output.isCancelled()         // true if the process was cancelled
output.checkSuccess(logger)  // logs stderr and returns false on non-zero exit code
```

### When to Use `CapturingProcessUtil` vs Manual Listener

| Scenario | Recommended approach |
|---|---|
| One-shot sync capture (version check, single query) | `CapturingProcessUtil.execAndGetOutput()` |
| Streaming output to UI / task window | `ProcessHandlerBuilderFactory` + `ProcessListener` |
| Long-running process with cancellation support | `ProcessHandlerBuilderFactory` + `ProcessListener` + `volatile ProcessHandler` |
| Need to distinguish stdout vs stderr separately | `CapturingProcessUtil` (use `getStdout()` / `getStderr()`) |

---

## `ProcessOutputType` Constants

```java
ProcessOutputType.SYSTEM   // IDE internal messages (start/stop notifications)
ProcessOutputType.STDOUT   // standard output
ProcessOutputType.STDERR   // standard error
```

Coloured output creates sub-types of the base types, so always use the static
`isStdout()` / `isStderr()` helpers rather than identity comparison.

---

## What NOT to Do

```java
// ❌ Raw JVM process API
ProcessBuilder pb = new ProcessBuilder("arduino-cli", "compile");
pb.redirectErrorStream(true);
Process process = pb.start();
process.waitFor();

// ❌ Deprecated ProcessAdapter
handler.addProcessListener(new ProcessAdapter() { ... });

// ❌ Deprecated static factory
ProcessHandlerBuilder.create(commandLine).build();   // @Deprecated — use factory

// ❌ Wrong Key import for onTextAvailable parameter
import consulo.util.lang.Key;   // ❌ — class does not exist there
import consulo.util.dataholder.Key;   // ✅ correct

// ❌ Identity comparison for output type
outputType == ProcessOutputType.STDOUT   // misses coloured subtypes

// ❌ Manual listener/StringBuilder for synchronous one-shot capture — use CapturingProcessUtil instead
StringBuilder buf = new StringBuilder();
handler.addProcessListener(new ProcessListener() {
    @Override
    public void onTextAvailable(ProcessEvent event, Key outputType) { buf.append(event.getText()); }
});
handler.startNotify();
handler.waitFor(5000);
// parse buf.toString()   ← replace this entire block with CapturingProcessUtil.execAndGetOutput()
```
