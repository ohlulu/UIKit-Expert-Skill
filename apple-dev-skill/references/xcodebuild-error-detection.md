# xcodebuild Error Detection

## Problem

xcodebuild incremental builds can return **exit code 0** even when a source file has a compile error — if derived data already contains a valid product from a previous successful build. This means checking only `PIPESTATUS[0]` is insufficient.

## Required Pattern

Every xcodebuild wrapper script must use **dual failure detection**:

```bash
xcodebuild build ... 2>&1 | tee /tmp/xc-build.log | grep -E "error:|\*\*" || true

EXIT=${PIPESTATUS[0]}
if [ $EXIT -ne 0 ] || grep -q "BUILD FAILED" /tmp/xc-build.log; then
  echo "BUILD FAILED"
  exit 1
fi
```

Both conditions are necessary:
- `PIPESTATUS[0]` catches most failures
- `grep BUILD FAILED` catches incremental build edge cases where exit code is 0

## Consistency Rule

All `xc-*.sh` scripts (`xc-build.sh`, `xc-build-run.sh`, `xc-test.sh`) must share the same error detection logic. When updating one, audit all others. A script that only checks exit code will eventually let a broken build through — especially during rapid iteration with `xc-build-run.sh` where the focus is on launching, not verifying.

## Also Include

- `CODE_SIGNING_ALLOWED=NO` — prevents signing errors from masking real build failures in CI and local scripts
- Consistent `XC_CONFIG` — all scripts should source the same `xc-env.sh` for scheme, destination, and configuration
