# CLAUDE.md - ViewDistanceTweaks Developer Guide

This document provides comprehensive information about the ViewDistanceTweaks codebase for AI assistants and developers.

## Project Overview

**ViewDistanceTweaks** is a Paper (Minecraft server) plugin that dynamically manages per-world simulation distance and view distance to optimize server performance. The plugin can operate in three modes:
- **Proactive**: Adjusts based on chunk count thresholds
- **Reactive**: Adjusts based on server MSPT (milliseconds per tick)
- **Mixed**: Combines both approaches, prioritizing decreases over increases

**Repository**: https://github.com/froobynooby/ViewDistanceTweaks
**Plugin Page**: https://www.spigotmc.org/resources/75164/
**Current Version**: 1.5.7
**Java Version**: 21
**Target API**: Paper 1.21-R0.1-SNAPSHOT

## Contributing Guidelines

**IMPORTANT**: This project follows standardized contributing guidelines maintained in `CONTRIBUTING.md`.

**Source of Truth**: https://denpaio.github.io/CONTRIBUTING.md

**Before any development work**:
1. Verify `CONTRIBUTING.md` is up-to-date with the source:
   ```bash
   curl -s https://denpaio.github.io/CONTRIBUTING.md | diff CONTRIBUTING.md -
   ```
2. If outdated, sync to latest version:
   ```bash
   curl -o CONTRIBUTING.md https://denpaio.github.io/CONTRIBUTING.md
   ```
3. Submit a PR with the updated file if changes are detected

**Key Standards from CONTRIBUTING.md**:
- **Java Style**: Follow [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html)
- **Commit Messages**: Use [Conventional Commits](https://www.conventionalcommits.org/) specification
  - Format: `type(scope): description`
  - Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, etc.
- **Documentation**: English preferred, self-documenting code over redundant comments
- **Testing**: Ensure all tests pass before committing

## Repository Structure

```
ViewDistanceTweaks/
├── .github/
│   └── workflows/
│       └── build.yml              # CI/CD workflow for building the plugin
├── gradle/                        # Gradle wrapper files
├── CLAUDE.md                      # Developer guide for AI assistants
├── CONTRIBUTING.md                # Contributing guidelines (synced from source)
├── src/
│   └── main/
│       ├── java/com/froobworld/viewdistancetweaks/
│       │   ├── ViewDistanceTweaks.java          # Main plugin class
│       │   ├── HookManager.java                  # Manages event hooks
│       │   ├── TaskManager.java                  # Manages scheduled tasks
│       │   ├── command/                          # Command implementations
│       │   │   ├── VdtCommand.java              # Main /vdt command handler
│       │   │   ├── ReloadCommand.java           # /vdt reload subcommand
│       │   │   ├── SetCommand.java              # /vdt set subcommands
│       │   │   └── StatusCommand.java           # /vdt status subcommand
│       │   ├── config/
│       │   │   └── VdtConfig.java               # Configuration management
│       │   ├── hook/
│       │   │   ├── tick/                        # Tick-related hooks
│       │   │   │   ├── TickHook.java
│       │   │   │   ├── PaperTickHook.java
│       │   │   │   └── SpigotTickHook.java
│       │   │   └── viewdistance/                # View/simulation distance hooks
│       │   │       ├── ViewDistanceHook.java
│       │   │       ├── SimulationDistanceHook.java
│       │   │       ├── PaperViewDistanceHook.java
│       │   │       └── PaperSimulationDistanceHook.java
│       │   ├── limiter/                         # Core distance management
│       │   │   ├── ClientViewDistanceManager.java
│       │   │   ├── ManualViewDistanceManager.java
│       │   │   ├── ViewDistanceChangeTask.java
│       │   │   ├── ViewDistanceClamper.java
│       │   │   ├── ViewDistanceLimiter.java
│       │   │   └── adjustmentmode/              # Distance adjustment algorithms
│       │   │       ├── AdjustmentMode.java      # Interface for adjustment modes
│       │   │       ├── AdjustmentModes.java     # Factory and utilities
│       │   │       ├── BaseAdjustmentMode.java
│       │   │       ├── MixedAdjustmentMode.java
│       │   │       ├── ProactiveAdjustmentMode.java
│       │   │       └── ReactiveAdjustmentMode.java
│       │   ├── metrics/
│       │   │   └── VdtMetrics.java              # bStats integration
│       │   ├── placeholder/                     # PlaceholderAPI integration
│       │   │   ├── VdtExpansion.java
│       │   │   └── handlers/                    # Placeholder handlers
│       │   └── util/                            # Utility classes
│       │       ├── ChunkCounter.java
│       │       ├── StandardChunkCounter.java
│       │       ├── NoTickChunkCounter.java
│       │       ├── CommandUtils.java
│       │       ├── MsptTracker.java
│       │       ├── TpsTracker.java
│       │       ├── PreferenceChooser.java
│       │       ├── RectangleUnionAreaFinder.java
│       │       └── ViewDistanceUtils.java
│       └── resources/
│           ├── plugin.yml                       # Plugin metadata & permissions
│           └── resources/
│               ├── config.yml                   # Default configuration
│               └── config-patches/              # Config update patches (1-8)
├── build.gradle.kts                             # Build configuration
├── settings.gradle                              # Gradle settings
├── gradlew / gradlew.bat                        # Gradle wrapper scripts
├── LICENSE                                      # License file
└── README.md                                    # Basic build instructions
```

## Key Components & Architecture

### 1. Main Plugin Class (`ViewDistanceTweaks.java`)

The entry point that:
- Validates Spigot/Paper environment (`org.spigotmc.SpigotConfig` check)
- Loads configuration via `VdtConfig`
- Initializes managers: `ClientViewDistanceManager`, `HookManager`, `TaskManager`
- Registers commands
- Integrates with PlaceholderAPI (if available)
- Sets up bStats metrics

**Initialization Order** (`onEnable`):
1. Environment validation (Spigot check)
2. Configuration loading
3. Enabled check (plugin must be explicitly enabled in config)
4. Client view distance manager initialization
5. Hook manager initialization
6. Task manager initialization
7. Command registration
8. Metrics initialization
9. PlaceholderAPI integration

### 2. Configuration System (`config/VdtConfig.java`)

Uses the **NabConfiguration** library (external dependency) for configuration management.

**Key Configuration Sections**:
- `enabled`: Master switch; plugin disables itself until set to true
- `adjustment-mode`: proactive | reactive | mixed
- `proactive-mode-settings`: Chunk count targets
- `reactive-mode-settings`: MSPT thresholds and prediction
- `world-settings`: Per-world configuration (simulation/view distance limits, chunk weights, exclusions)
- `ticks-per-check`: How often to evaluate distance changes (default: 600 ticks = 30 seconds)
- `start-up-delay`: Ticks to wait before starting checks (default: 2400 = 2 minutes)
- `passed-checks-for-increase/decrease`: Hysteresis for distance changes

### 3. Adjustment Modes (`limiter/adjustmentmode/`)

**Three strategies for managing distances**:

- **ProactiveAdjustmentMode**: Maintains chunk counts below thresholds
  - Monitors global ticking chunk count
  - Monitors non-ticking chunk count (view distance - simulation distance)
  - Adjusts distances to stay at target values

- **ReactiveAdjustmentMode**: Responds to server MSPT
  - Increases distance when MSPT < increase threshold (default: 40ms)
  - Decreases distance when MSPT > decrease threshold (default: 47ms)
  - Includes MSPT prediction to avoid oscillation
  - Optionally manages view distance at a target ratio to simulation distance

- **MixedAdjustmentMode**: Combines both, prioritizing decreases

All modes implement the `AdjustmentMode` interface and may extend `BaseAdjustmentMode`. The `AdjustmentModes` factory builds the configured mode.

### 4. Hook System (`HookManager.java` + `hook/`)

Manages event listeners for:
- **Tick hooks**: Monitoring server tick performance / MSPT (`TickHook` with `PaperTickHook` and `SpigotTickHook` implementations)
- **View distance hooks**: Reading/writing view and simulation distance (`ViewDistanceHook`, `SimulationDistanceHook` with Paper-specific implementations)

### 5. Task System (`TaskManager.java`)

Schedules repeating tasks that:
- Check if distance adjustments are needed (every `ticks-per-check`)
- Apply hysteresis logic (requiring N consecutive checks before changing)
- Execute distance changes via `ViewDistanceChangeTask`

### 6. Distance Management (`limiter/`)

- **ClientViewDistanceManager**: Manages client-specific view distance settings
- **ManualViewDistanceManager**: Handles manual overrides
- **ViewDistanceLimiter**: Core logic for calculating and applying distance limits
- **ViewDistanceClamper**: Ensures distances stay within configured min/max bounds
- **ViewDistanceChangeTask**: Executes distance changes

### 7. Utilities (`util/`)

- **ChunkCounter** / **StandardChunkCounter** / **NoTickChunkCounter**: Count chunks loaded by players (with overlap detection)
- **MsptTracker**: Tracks milliseconds per tick with configurable collection period
- **TpsTracker**: Tracks ticks per second
- **ViewDistanceUtils**: Helper methods for distance calculations
- **PreferenceChooser**: Utility for selecting which worlds to adjust
- **RectangleUnionAreaFinder**: Computes non-overlapping chunk area
- **CommandUtils**: Shared command helpers

### 8. Commands (`command/`)

- **/vdt**: Base command (alias `/viewdistancetweaks`)
- **/vdt reload**: Reload configuration
- **/vdt status**: Show current distance settings and performance metrics
- **/vdt simulationdistance <world> <distance>**: Manually set simulation distance
- **/vdt viewdistance <world> <distance>**: Manually set view distance

### 9. Integration (`placeholder/`, `metrics/`)

- **PlaceholderAPI** (soft dependency): Provides placeholders for use in other plugins via `VdtExpansion` and handlers in `placeholder/handlers/`
- **bStats**: Collects anonymous usage statistics via `VdtMetrics`

## Development Workflows

### Building the Plugin

**Dependencies**:
1. First install `NabConfiguration` to Maven local:
   ```bash
   git clone https://github.com/froobynooby/nab-configuration
   cd nab-configuration
   ./gradlew clean install
   ```

2. Build ViewDistanceTweaks:
   ```bash
   ./gradlew clean shadowJar
   ```

3. Output location: `build/libs/ViewDistanceTweaks-1.5.7.jar`

**Gradle Tasks**:
- `./gradlew clean`: Clean build directory
- `./gradlew build`: Compile and run tests
- `./gradlew shadowJar`: Build fat JAR with shaded dependencies

**Build Plugins**:
- `io.papermc.paperweight.userdev` (1.7.1) — Paper dev bundle for NMS access
- `io.github.goooler.shadow` (8.1.7) — shaded fat JAR

**Shaded Dependencies** (relocated to avoid conflicts):
- `com.froobworld.nabconfiguration` → `com.froobworld.viewdistancetweaks.lib.nabconfiguration`
- `org.joor` → `com.froobworld.viewdistancetweaks.lib.joor`
- `org.bstats` → `com.froobworld.viewdistancetweaks.lib.bstats`

### CI/CD (GitHub Actions)

**Workflow** (`.github/workflows/build.yml`):
1. Set up JDK 8 for NabConfiguration
2. Check out and install NabConfiguration
3. Switch to JDK 21
4. Check out ViewDistanceTweaks
5. Validate Gradle wrapper
6. Build with `shadowJar`
7. Upload artifact

**Triggered on**: Push and pull requests to any branch

### Testing

- No automated test suite currently exists
- Manual testing required on a Paper server
- Test with different adjustment modes and configurations
- Monitor MSPT, TPS, and chunk counts during testing

## Coding Conventions

**See `CONTRIBUTING.md` for complete style guidelines and standards.**

### Java Standards

- **Java Version**: 21 (language level and toolchain)
- **API Target**: Paper 1.21-R0.1-SNAPSHOT (`plugin.yml` declares `api-version: 1.20`)
- **Encoding**: UTF-8 for all source files and Javadoc
- **Style Guide**: Follow [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html) as specified in `CONTRIBUTING.md`

### Code Organization

- **Package Structure**: Feature-based packages (command, config, hook, limiter, etc.)
- **Naming**: Descriptive class names following Java conventions
- **Main Plugin Class**: Should be lightweight, delegating to managers

### Configuration Patterns

- Use **NabConfiguration** library for config management
- Support per-world overrides with defaults
- Include descriptive comments in default `config.yml`
- Version config schema (current: version 9; patches in `config-patches/` go up to `8.patch`)

### Hook/Listener Patterns

- Centralize hook registration in `HookManager`
- Separate tick hooks from view distance hooks
- Provide Paper-specific implementations behind shared interfaces

### Distance Adjustment Patterns

- All adjustment modes implement `AdjustmentMode` interface
- Use `AdjustmentModes` factory for creating mode instances
- Apply hysteresis to prevent oscillation
- Respect per-world min/max bounds

### Manager Pattern

- Core functionality split into managers: `HookManager`, `TaskManager`, `ClientViewDistanceManager`
- Managers initialized in specific order in `ViewDistanceTweaks.onEnable()`
- Managers provide getter methods for access from other components

## Important Considerations for AI Assistants

### 0. Contributing Guidelines Compliance

**CRITICAL**: Before starting any work, verify `CONTRIBUTING.md` is up-to-date with the source of truth:
```bash
curl -s https://denpaio.github.io/CONTRIBUTING.md | diff CONTRIBUTING.md -
```

If outdated, MUST update it first via PR before proceeding with other changes. All code must follow:
- [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html)
- [Conventional Commits](https://www.conventionalcommits.org/) specification for commit messages
- Self-documenting code principles
- Testing requirements as specified in `CONTRIBUTING.md`

### 1. Dependency Management

**CRITICAL**: This plugin depends on `NabConfiguration` which must be installed to Maven local before building. This is NOT available in Maven Central.

When helping with build issues:
- Always ensure NabConfiguration is installed first
- Check Maven local cache if build fails with missing dependency

### 2. Configuration Schema

The config has a **version** field that tracks schema changes. When modifying configuration:
- Increment the version number if schema changes
- Add migration patches in `src/main/resources/resources/config-patches/`
- Test with existing config files to ensure backward compatibility

### 3. Performance Sensitivity

This plugin directly impacts server performance. When modifying distance adjustment logic:
- Avoid adding expensive operations in tick hooks
- Be mindful of chunk counting overhead
- Test with high player counts
- Consider the `exclude-overlap` setting for chunk counting

### 4. Hysteresis Logic

The plugin uses hysteresis (requiring N consecutive checks) to prevent oscillation. When modifying adjustment logic:
- Preserve or enhance hysteresis mechanisms
- Test that distances don't rapidly fluctuate
- Consider the `passed-checks-for-increase` and `passed-checks-for-decrease` config values

### 5. Per-World Configuration

All settings can be overridden per-world. When adding new features:
- Support per-world configuration where appropriate
- Fall back to default world settings
- Allow worlds to be excluded from management

### 6. PlaceholderAPI Integration

The plugin provides placeholders (optional dependency). When adding metrics:
- Consider exposing them via PlaceholderAPI
- Add handlers in `placeholder/handlers/`
- Register in `VdtExpansion`

### 7. Reload Safety

The plugin supports `/vdt reload` for config changes. When adding features:
- Ensure proper cleanup of old state
- Reinitialize components safely in `ViewDistanceTweaks.reload()`
- Test reload without full server restart

### 8. Git Workflow

**Branch Naming**: When working on features, use branches prefixed with `claude/` and include session ID
**Commits**: Follow [Conventional Commits](https://www.conventionalcommits.org/) specification (see `CONTRIBUTING.md`)
  - Format: `type(scope): description`
  - Example: `feat(limiter): add reactive mode for view distance`, `fix(config): correct default MSPT threshold`
  - Common types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`
**Build Verification**: Ensure GitHub Actions workflow passes before merging

### 9. Paper API Usage

- Use Paper-specific APIs where available (not just Bukkit/Spigot)
- Paper provides enhanced chunk and world management APIs
- Target API version declared in `plugin.yml`: 1.20

### 10. Shadow JAR Configuration

Dependencies are shaded (relocated) to avoid conflicts. When adding dependencies:
- Add relocation rules in `build.gradle.kts` `shadowJar` task
- Use package prefix: `com.froobworld.viewdistancetweaks.lib.<dependency>`
- Test that relocated classes work correctly

## Common Tasks Guide

### Adding a New Adjustment Mode

1. Create class implementing `AdjustmentMode` in `limiter/adjustmentmode/`
2. Extend `BaseAdjustmentMode` if applicable
3. Add factory method in `AdjustmentModes`
4. Update config.yml with new mode option
5. Document behavior in config comments

### Adding a New Command

1. Create command class in `command/` package
2. Implement command logic and tab completion
3. Register in `VdtCommand` as subcommand
4. Add permission nodes in `plugin.yml`
5. Test with and without permission

### Adding Configuration Options

1. Add field to `VdtConfig.java`
2. Initialize with default value
3. Update `resources/config.yml` with new option and comments
4. If schema changes significantly, increment config version
5. Add migration patch if needed in `config-patches/`

### Adding Placeholders

1. Create handler in `placeholder/handlers/` implementing `PlaceholderHandler`
2. Register in `VdtExpansion`
3. Document placeholder format
4. Test with PlaceholderAPI installed and not installed

### Debugging Distance Changes

Key log points:
- `ViewDistanceTweaks.onEnable()`: Plugin initialization
- `TaskManager`: Task scheduling and execution
- `ViewDistanceLimiter`: Distance calculation logic
- `ViewDistanceChangeTask`: Actual distance application
- Enable `log-changes: true` in config for distance change logging

## File References by Feature

### Simulation Distance Management
- `ViewDistanceTweaks.java:43-48` - Manager initialization
- `limiter/ViewDistanceLimiter.java` - Core logic
- `limiter/adjustmentmode/*` - Adjustment algorithms

### Configuration System
- `config/VdtConfig.java` - Config class
- `resources/config.yml` - Default config
- `ViewDistanceTweaks.java:26-34` - Config loading

### Command System
- `command/VdtCommand.java` - Main command
- `command/StatusCommand.java` - Status display
- `command/SetCommand.java` - Manual distance setting
- `ViewDistanceTweaks.java:73-78` - Command registration

### Performance Tracking
- `util/MsptTracker.java` - MSPT measurement
- `util/TpsTracker.java` - TPS measurement
- `util/ChunkCounter.java` - Chunk counting

### Event Handling
- `HookManager.java` - Hook registration
- `hook/tick/*` - Tick performance hooks
- `hook/viewdistance/*` - Distance change hooks

## Version History Notes

- **1.5.7** (Current): Latest stable version
- Config version: 9
- Target: Paper 1.21
- Requires: Java 21

## External Resources

- **Paper API Docs**: https://jd.papermc.io/paper/1.21/
- **Bukkit API Docs**: https://hub.spigotmc.org/javadocs/bukkit/
- **PlaceholderAPI**: https://github.com/PlaceholderAPI/PlaceholderAPI
- **bStats**: https://bstats.org/
- **NabConfiguration**: https://github.com/froobynooby/nab-configuration

## Support & Contribution

- **Contributing**: See `CONTRIBUTING.md` for style guidelines and workflow
- **Issues**: Report on GitHub repository
- **Plugin Page**: https://www.spigotmc.org/resources/75164/
- **Author**: froobynooby
