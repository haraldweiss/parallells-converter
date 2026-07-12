# Parallels-to-UTM Converter Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a native macOS SwiftUI app that converts selected, shut-down Parallels PVMs into verified UTM QEMU VMs without modifying the source PVM.

**Architecture:** A Swift Package provides a SwiftUI shell, domain models, XML-based PVM inspection, QEMU process integration and UTM AppleScript automation. The app scans only an explicitly selected source directory, serializes conversion jobs, writes only below the selected destination directory, and emits structured progress and logs for the UI.

**Tech Stack:** Swift 6.4, SwiftUI, Foundation/XMLParser, XCTest, macOS `Process`, installed QEMU (`qemu-img`), installed UTM AppleScript interface.

## Global Constraints

- Native macOS SwiftUI; do not introduce Tauri, React, Electron, Homebrew dependencies or a server.
- Require macOS with UTM and `qemu-img` already installed; report missing dependencies before a job starts.
- Never run `prl_disk_tool merge` and never write to a source `.pvm` bundle.
- Resolve the only convertible image from `DiskDescriptor.xml`; do not enumerate and convert every `.hds` file.
- If the source volume rejects QEMU byte locking, use `qemu-img -U` only after the PVM has passed the shut-down/saved-state preflight.
- Process at most one VM at a time and make all generated paths descendants of the user-selected target directory.
- The existing 256-GiB reference PVM is an explicit manual acceptance test only; automated tests use small committed fixtures.

---

## File Structure

- `Package.swift` — macOS application executable and test target.
- `Sources/ParallelsUTMConverter/ParallelsUTMConverterApp.swift` — SwiftUI entry point.
- `Sources/ParallelsUTMConverter/Domain/VirtualMachine.swift` — immutable PVM, disk, host and job state models.
- `Sources/ParallelsUTMConverter/Infrastructure/PVMInspector.swift` — scans PVM packages and parses XML metadata.
- `Sources/ParallelsUTMConverter/Infrastructure/DependencyLocator.swift` — resolves QEMU and UTM executables.
- `Sources/ParallelsUTMConverter/Infrastructure/ProcessRunner.swift` — argument-safe subprocess execution, output streaming and cancellation.
- `Sources/ParallelsUTMConverter/Infrastructure/QemuImageService.swift` — preflight, QCOW2 conversion and image verification.
- `Sources/ParallelsUTMConverter/Infrastructure/UTMService.swift` — creates and exports a configured UTM QEMU VM through AppleScript.
- `Sources/ParallelsUTMConverter/Features/Converter/ConversionCoordinator.swift` — serial queue, lifecycle and cleanup policy.
- `Sources/ParallelsUTMConverter/Features/Converter/ConverterViewModel.swift` — `@MainActor` UI state and actions.
- `Sources/ParallelsUTMConverter/Features/Converter/ConverterView.swift` — folders, selection, preflight, progress and logs.
- `Tests/ParallelsUTMConverterTests/Fixtures/*.xml` — minimal PVM and disk descriptor fixtures.
- `Tests/ParallelsUTMConverterTests/*Tests.swift` — unit tests with fake process and file-system seams.
- `README.md` — setup, permissions, workflow and current limitations.

## Task 1: Swift package and testable application foundation

**Files:**
- Create: `Package.swift`
- Create: `Sources/ParallelsUTMConverter/ParallelsUTMConverterApp.swift`
- Create: `Sources/ParallelsUTMConverter/Domain/VirtualMachine.swift`
- Create: `Tests/ParallelsUTMConverterTests/VirtualMachineTests.swift`

**Interfaces:**
- Produces `PVMachine`, `PVMachineState`, `DiskImage`, `ConversionPhase`, and `ConversionJob` used by all remaining tasks.

- [ ] **Step 1: Write the failing model test**

```swift
func testJobStartsQueuedAndCanBecomeRunning() {
    var job = ConversionJob(machine: fixtureMachine)
    XCTAssertEqual(job.phase, .queued)
    job.phase = .converting(progress: 0.25)
    XCTAssertEqual(job.progress, 0.25)
}
```

- [ ] **Step 2: Run the test to confirm the package is absent**

Run: `swift test --filter VirtualMachineTests/testJobStartsQueuedAndCanBecomeRunning`

Expected: FAIL because no package or model exists.

- [ ] **Step 3: Add the package and minimal models**

```swift
public enum ConversionPhase: Equatable, Sendable {
    case queued, preflighting, converting(progress: Double), registeringUTM, verifying, succeeded(URL), failed(String), cancelled
}

public struct ConversionJob: Identifiable, Sendable {
    public let id = UUID()
    public let machine: PVMachine
    public var phase: ConversionPhase = .queued
    public var progress: Double { if case let .converting(value) = phase { value } else { 0 } }
}
```

- [ ] **Step 4: Add the minimal SwiftUI app shell**

```swift
@main
struct ParallelsUTMConverterApp: App {
    var body: some Scene { WindowGroup { ConverterView() } }
}
```

- [ ] **Step 5: Run the model suite and build**

Run: `swift test && swift build`

Expected: PASS; the executable target compiles on macOS.

- [ ] **Step 6: Commit**

```bash
git add Package.swift Sources Tests
git commit -m "feat: scaffold native converter app"
```

## Task 2: PVM inspection and active disk resolution

**Files:**
- Create: `Sources/ParallelsUTMConverter/Infrastructure/PVMInspector.swift`
- Create: `Tests/ParallelsUTMConverterTests/PVMInspectorTests.swift`
- Create: `Tests/ParallelsUTMConverterTests/Fixtures/DiskDescriptor.xml`
- Create: `Tests/ParallelsUTMConverterTests/Fixtures/config.pvs`

**Interfaces:**
- Consumes: `PVMachine`, `DiskImage` from Task 1.
- Produces: `PVMInspecting.inspect(pvmURL:) async throws -> PVMachine` and `scan(sourceURL:) async throws -> [PVMachine]`.

- [ ] **Step 1: Write failing descriptor tests**

```swift
func testInspectorUsesOnlyImageNamedByDescriptor() async throws {
    let machine = try await inspector.inspect(pvmURL: fixturePVM)
    XCTAssertEqual(machine.primaryDisk.url.lastPathComponent,
                   "Windows 11-0.hdd.0.{active}.hds")
}

func testInspectorRejectsDescriptorWithoutImage() async {
    do {
        _ = try await inspector.inspect(pvmURL: brokenPVM)
        XCTFail("Expected invalid descriptor to be rejected")
    } catch {
        XCTAssertEqual(error as? PVMInspectionError, .missingActiveDisk)
    }
}
```

- [ ] **Step 2: Run the focused suite**

Run: `swift test --filter PVMInspectorTests`

Expected: FAIL because `PVMInspector` is not defined.

- [ ] **Step 3: Implement focused XML parsing**

```swift
protocol PVMInspecting: Sendable {
    func scan(sourceURL: URL) async throws -> [PVMachine]
    func inspect(pvmURL: URL) async throws -> PVMachine
}

// Parse DiskDescriptor.xml StorageData/Storage/Image/File exactly once.
// Validate standardizedPath has the disk bundle as its parent; reject traversal.
```

Parse `config.pvs` for VM name, RAM, CPU count and network presence. Set `requiresUnsafeShare` only from a typed preflight result, not from the filename.

- [ ] **Step 4: Run tests and verify real inspection is read-only**

Run: `swift test --filter PVMInspectorTests`

Expected: PASS.

Run: `swift run ParallelsUTMConverter --inspect '/Volumes/MAC/Windows on ARM M1/Windows 11 (1).pvm'`

Expected: reports the active `5fba…hds`, 8 CPUs and 8192 MiB; no source modification. Add the `--inspect` developer-only command in the executable target.

- [ ] **Step 5: Commit**

```bash
git add Sources Tests
git commit -m "feat: inspect Parallels PVM metadata safely"
```

## Task 3: Dependency and conversion preflight

**Files:**
- Create: `Sources/ParallelsUTMConverter/Infrastructure/DependencyLocator.swift`
- Create: `Sources/ParallelsUTMConverter/Infrastructure/ProcessRunner.swift`
- Create: `Sources/ParallelsUTMConverter/Infrastructure/QemuImageService.swift`
- Create: `Tests/ParallelsUTMConverterTests/QemuImageServiceTests.swift`

**Interfaces:**
- Consumes: `PVMachine` and `DiskImage`.
- Produces: `ConversionPreflight` with `qemuURL`, `utmURL`, `freeBytes`, `useUnsafeShare`, and blocking `issues`.

- [ ] **Step 1: Write failing preflight tests**

```swift
func testUnsupportedByteLockRequestsUnsafeReadOnlyShare() async throws {
    runner.stubbedResult = .failure(stderr: "Failed to lock byte 100: Operation not supported")
    let result = try await service.preflight(machine: fixtureMachine, destination: destination)
    XCTAssertTrue(result.useUnsafeShare)
}

func testMissingQemuBlocksPreflight() async throws {
    locator.qemuURL = nil
    let result = try await service.preflight(machine: fixtureMachine, destination: destination)
    XCTAssertEqual(result.issues, [.missingQemu])
}
```

- [ ] **Step 2: Run focused tests**

Run: `swift test --filter QemuImageServiceTests`

Expected: FAIL because the services do not exist.

- [ ] **Step 3: Implement non-shell subprocess boundary**

```swift
protocol ProcessRunning: Sendable {
    func run(executable: URL, arguments: [String], workingDirectory: URL?, onOutput: @Sendable (String) -> Void) async throws -> ProcessResult
}

let qemuCandidates = ["/opt/homebrew/bin/qemu-img", "/usr/local/bin/qemu-img", "/usr/bin/qemu-img"]
let utmURL = URL(fileURLWithPath: "/Applications/UTM.app")
```

Use `Process.executableURL` and `arguments`, never concatenate a shell command. Run `qemu-img info` first. Detect only the exact lock-support error; retry the read-only info command with `-U` only if saved-state checks pass. Calculate destination free space with `.volumeAvailableCapacityForImportantUsageKey` and require the image's physical size plus 10% headroom.

- [ ] **Step 4: Run tests and live dependency check**

Run: `swift test --filter QemuImageServiceTests && /opt/homebrew/bin/qemu-img --version`

Expected: tests PASS and installed QEMU version prints.

- [ ] **Step 5: Commit**

```bash
git add Sources Tests
git commit -m "feat: preflight QEMU and destination safely"
```

## Task 4: QCOW2 conversion, progress, cancellation and verification

**Files:**
- Modify: `Sources/ParallelsUTMConverter/Infrastructure/QemuImageService.swift`
- Create: `Tests/ParallelsUTMConverterTests/ConversionCommandTests.swift`

**Interfaces:**
- Consumes: successful `ConversionPreflight`.
- Produces: `convert(machine:preflight:destination:onProgress:) async throws -> URL` and `verify(qcowURL:) async throws`.

- [ ] **Step 1: Write command-construction tests**

```swift
func testConversionUsesActiveDiskAndUnsafeShareWhenRequired() {
    let arguments = service.convertArguments(input: activeDisk, output: output, useUnsafeShare: true)
    XCTAssertEqual(arguments, ["convert", "-U", "-f", "parallels", "-O", "qcow2", "-p", activeDisk.path, output.path])
}

func testConversionNeverUsesMergeOrSourceBundleAsOutput() {
    XCTAssertFalse(arguments.contains("merge"))
    XCTAssertNotEqual(URL(fileURLWithPath: arguments.last!), fixturePVM)
}
```

- [ ] **Step 2: Run the test**

Run: `swift test --filter ConversionCommandTests`

Expected: FAIL because conversion argument construction is absent.

- [ ] **Step 3: Implement conversion lifecycle**

```swift
func convert(...) async throws -> URL {
    try FileManager.default.createDirectory(at: stagingURL, withIntermediateDirectories: true)
    defer { if Task.isCancelled { try? FileManager.default.removeItem(at: stagingURL) } }
    let result = try await runner.run(executable: qemuURL, arguments: convertArguments(...), workingDirectory: nil, onOutput: parseProgress)
    guard result.exitCode == 0 else { throw ConversionError.qemuFailed(result.stderr) }
    try await verify(qcowURL: outputURL)
    return outputURL
}
```

Parse only the percentage output from `qemu-img -p`; append all stdout/stderr to a job-local log. On failure leave a `.failed` marker with the log under the destination, but never delete user-created destination content.

- [ ] **Step 4: Run tests**

Run: `swift test --filter ConversionCommandTests`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add Sources Tests
git commit -m "feat: convert Parallels disks to verified qcow2"
```

## Task 5: UTM registration and export

**Files:**
- Create: `Sources/ParallelsUTMConverter/Infrastructure/UTMService.swift`
- Create: `Tests/ParallelsUTMConverterTests/UTMServiceTests.swift`

**Interfaces:**
- Consumes: `PVMachine` and verified QCOW2 URL.
- Produces: `UTMRegistering.createAndExport(machine:qcowURL:destination:) async throws -> URL`.

- [ ] **Step 1: Write failing AppleScript-generation tests**

```swift
func testScriptCreatesArmQemuVMWithConvertedDrive() throws {
    let script = try service.creationScript(machine: armMachine, qcowURL: diskURL, exportURL: outputURL)
    XCTAssertTrue(script.contains("backend:qemu"))
    XCTAssertTrue(script.contains("architecture:\"aarch64\""))
    XCTAssertTrue(script.contains("memory:8192"))
    XCTAssertTrue(script.contains("cpu cores:8"))
    XCTAssertTrue(script.contains("interface:VirtIO"))
}
```

- [ ] **Step 2: Run focused test**

Run: `swift test --filter UTMServiceTests`

Expected: FAIL because `UTMService` does not exist.

- [ ] **Step 3: Implement UTM AppleScript process**

```applescript
tell application "UTM"
  set diskFile to POSIX file "__QCOW_PATH__"
  set exportFile to POSIX file "__UTM_PATH__"
  set vm to make new virtual machine with properties {backend:qemu, configuration:{name:"__NAME__", architecture:"aarch64", memory:8192, cpu cores:8, hypervisor:true, uefi:true, drives:{{source:diskFile, interface:VirtIO}}, network interfaces:{{mode:shared}}}}
  export vm to exportFile
end tell
```

Escape AppleScript string literals before interpolation and invoke `/usr/bin/osascript` through `ProcessRunner`. For x86 guests choose `x86_64`; set `hypervisor` to `false` on architecture mismatch. If export fails, preserve the QCOW2 and job log and surface an actionable UTM error.

- [ ] **Step 4: Run tests and a harmless UTM availability check**

Run: `swift test --filter UTMServiceTests && /Applications/UTM.app/Contents/MacOS/utmctl --help`

Expected: PASS; UTM CLI help prints. Do not create a real VM in this step.

- [ ] **Step 5: Commit**

```bash
git add Sources Tests
git commit -m "feat: create UTM VMs from converted disks"
```

## Task 6: Serial coordinator and SwiftUI workflow

**Files:**
- Create: `Sources/ParallelsUTMConverter/Features/Converter/ConversionCoordinator.swift`
- Create: `Sources/ParallelsUTMConverter/Features/Converter/ConverterViewModel.swift`
- Create: `Sources/ParallelsUTMConverter/Features/Converter/ConverterView.swift`
- Create: `Tests/ParallelsUTMConverterTests/ConversionCoordinatorTests.swift`

**Interfaces:**
- Consumes: inspector and service protocols from Tasks 2–5.
- Produces: `ConverterViewModel.selectSource()`, `selectDestination()`, `scan()`, `preflightSelected()`, `startSelected()` and `cancelCurrent()`.

- [ ] **Step 1: Write failing serial-execution test**

```swift
func testCoordinatorDoesNotStartSecondMachineBeforeFirstFinishes() async throws {
    let jobs = try await coordinator.convert([first, second], destination: destination)
    XCTAssertEqual(fakeConverter.startedMachineIDs, [first.id, second.id])
    XCTAssertTrue(fakeConverter.startedSecondAfterFirstCompletion)
    XCTAssertEqual(jobs.map(\.phase), [.succeeded(firstOutput), .succeeded(secondOutput)])
}
```

- [ ] **Step 2: Run focused test**

Run: `swift test --filter ConversionCoordinatorTests`

Expected: FAIL because no coordinator exists.

- [ ] **Step 3: Implement coordinator and view model**

```swift
for machine in selectedMachines {
    try Task.checkCancellation()
    await update(machine, phase: .preflighting)
    let preflight = try await qemu.preflight(machine: machine, destination: destination)
    await update(machine, phase: .converting(progress: 0))
    let qcow = try await qemu.convert(machine: machine, preflight: preflight, destination: staging, onProgress: progress)
    await update(machine, phase: .registeringUTM)
    let output = try await utm.createAndExport(machine: machine, qcowURL: qcow, destination: destination)
    await update(machine, phase: .succeeded(output))
}
```

Use `NSOpenPanel` directory selection in the view model. The view presents source/destination, selectable rows, blocking preflight issues, a disabled start button until both folders and a selected convertible VM exist, progress bars and an expandable text log. Require an explicit confirmation alert before `startSelected()`.

- [ ] **Step 4: Run test and application build**

Run: `swift test --filter ConversionCoordinatorTests && swift build`

Expected: PASS and app target compiles.

- [ ] **Step 5: Commit**

```bash
git add Sources Tests
git commit -m "feat: add converter workflow UI"
```

## Task 7: Documentation and explicit acceptance conversion

**Files:**
- Create: `README.md`
- Modify: `docs/superpowers/specs/2026-07-12-parallels-to-utm-converter-design.md`

**Interfaces:**
- Documents the UI workflow and the source-safety guarantee delivered by Tasks 1–6.

- [ ] **Step 1: Write README acceptance checklist**

```markdown
1. Quit the Parallels VM completely; do not suspend it.
2. Start the app, select the folder containing `.pvm` packages and an empty/adequately sized target folder.
3. Select one VM, review the preflight, then confirm conversion.
4. Open the resulting `.utm` in UTM and confirm Windows boots before deleting any source.
```

- [ ] **Step 2: Run the complete automated verification**

Run: `swift test && swift build`

Expected: PASS.

- [ ] **Step 3: Request explicit user confirmation before live conversion**

Use the source path `/Volumes/MAC/Windows on ARM M1/Windows 11 (1).pvm` only after the user explicitly confirms the target folder and that the VM is shut down. Do not embed this personal path in production code or tests.

- [ ] **Step 4: Perform and record manual acceptance after permission**

Expected: the target contains a UTM package, QEMU verification succeeds, and UTM can open the VM. Record the version of QEMU/UTM and any Windows driver/activation follow-up in `README.md`.

- [ ] **Step 5: Commit**

```bash
git add README.md docs
git commit -m "docs: add converter setup and acceptance guide"
```

## Plan self-review

- Spec coverage: Tasks 1–6 cover native UI, selected PVM scanning, settings transfer, source safety, sequential processing, QEMU/UTM availability, lock fallback, progress, logs and verification. Task 7 covers user-facing operation and the permitted real-PVM check.
- Placeholder scan: No unresolved markers; every task names exact paths, interfaces, tests and verification commands.
- Type consistency: `PVMachine`, `ConversionJob`, `ConversionPreflight`, `ProcessRunning`, `PVMInspecting`, and `UTMRegistering` are defined before their consumers.
