# Makefile

Reference template: [Makefile.template](Makefile.template)

## Design Principles

1. **`help` is the front door** — `.DEFAULT_GOAL := help`; newcomer types `make` and sees all targets
2. **DRY `xc` macro** — one macro for all xcodebuild calls; handles log capture, result summary, failure diagnostics
3. **Granular test targets** — `test` (fastest), `test-<module>`, `test-all` (CI)
4. **Comments explain WHY** — not what the target name already says
5. **Self-documenting** — `##` comments extracted by `awk` in `help` target
6. **Version management** — `bump VERSION=x.y` and `bump-build`; detect source (xcconfig vs Project.swift)
7. **Release delegates to scripts/** — Makefile is the menu; scripts/ is the kitchen
8. **Close Xcode before regeneration** — AppleScript closes only this project's window
9. **Complete `.PHONY`** — every non-file target listed

## Simulator Destination

Always pin both device model **and** OS version in the `SIM` variable — otherwise xcodebuild picks whichever runtime is available, which can drift between machines or after Xcode updates:

```makefile
SIM := platform=iOS Simulator,name=iPhone 17 Pro,OS=26.4
```

When generating a new Makefile or build script that includes a simulator destination, **ask the user which device and OS version to target**. List available simulators with `xcrun simctl list devices available` to offer concrete choices.

## Simulator Run Target

Always provide a `run` target that builds, installs, and launches in one command. **Never hardcode the bundle ID** — debug builds often append `.debug` or other suffixes. Read it from the built `.app`:

```makefile
## Build + install + launch on simulator
run: build
	@APP=$(DD)/Build/Products/Debug-iphonesimulator/$(SCHEME).app; \
	BID=$$(/usr/libexec/PlistBuddy -c 'Print :CFBundleIdentifier' "$$APP/Info.plist"); \
	xcrun simctl install booted "$$APP" && \
	xcrun simctl launch booted "$$BID"
```

Key rules:
- Bundle ID **must** come from `PlistBuddy` reading the build artifact's `Info.plist` — never guess or hardcode
- Use `booted` as the device target — avoids needing to know the exact UDID
- If multiple simulators are booted, `booted` picks the first one; shut down extras or target by UDID
- `make run` is the single command agents and developers should use — no manual `simctl` sequences

## Warnings Target

The `xc` macro redirects all output to a log file — warnings are captured but never shown on screen. Additionally, CLI builds typically use project-local DerivedData (`.derivedData/`) while Xcode IDE uses `~/Library/Developer/Xcode/DerivedData/<project>-<hash>`. Different incremental caches can produce different diagnostics, so CLI may show 0 warnings while the IDE shows many.

Solution: a dedicated `warnings` target that clean-builds using Xcode's DerivedData:

```makefile
# Xcode IDE DerivedData — resolved at runtime
XCODE_DD = $(shell find ~/Library/Developer/Xcode/DerivedData -maxdepth 1 -name '$(SCHEME)-*' -type d 2>/dev/null | head -1)

## Show Swift warnings (uses Xcode's DerivedData for parity with IDE)
warnings:
	@if [ -z "$(XCODE_DD)" ]; then \
	  echo "⚠️  No Xcode DerivedData found. Open project in Xcode first."; \
	  exit 1; \
	fi; \
	mkdir -p $(LOGDIR); \
	echo "⏳ clean build (Xcode DerivedData) …"; \
	xcodebuild clean build \
	  -workspace $(WORKSPACE) \
	  -scheme $(SCHEME) \
	  -destination '$(SIM)' \
	  -derivedDataPath "$(XCODE_DD)" \
	  -configuration Debug \
	  > $(LOGDIR)/warnings.log 2>&1 || true; \
	grep 'warning:' $(LOGDIR)/warnings.log \
	  | grep -v 'appintentsmetadata\|libtool' \
	  | sort -u; \
	COUNT=$$(grep 'warning:' $(LOGDIR)/warnings.log \
	  | grep -v 'appintentsmetadata\|libtool' \
	  | wc -l | tr -d ' '); \
	echo ""; echo "✅  $$COUNT warning(s)"
```

Clean build is essential — incremental builds cache stale diagnostics and may report 0 warnings even when issues exist. Adjust the `grep -v` noise filter for your project's specific non-actionable warnings.

## Adaptation Checklist

- [ ] Version source: xcconfig (`MARKETING_VERSION`) or Project.swift
- [ ] Test structure: Xcode scheme tests, SPM package tests, or both
- [ ] Generator: Tuist → add `generate` target; none → skip
- [ ] Release flow: App Store → full pipeline; framework → tag only
- [ ] Simulator destination: device + OS version pinned (ask user)
- [ ] Project-specific tooling (codegen, fixtures, linting)
