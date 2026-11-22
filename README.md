# Reproduction for rules_python#4556

Reproduces <https://github.com/bazel-contrib/rules_python/issues/4556>.
`all_whl_requirements` fails analysis on a platform where a platform-exclusive package is unreachable (rules_python 2.5.1, `pip.parse(uv_lock = ...)`).

## Setup

- `pyproject.toml` depends on `ipykernel` and locks for Linux and macOS in one universal `uv.lock`.
- The lock contains the edge `{ name = "appnope", marker = "sys_platform != 'linux'" }`, so `appnope` is macOS-only.
- `MODULE.bazel` calls `pip.parse` with `target_platforms = ["linux_aarch64", "linux_x86_64", "osx_aarch64"]`.
- `//:all_whls` is a `filegroup` whose `srcs` is `all_whl_requirements`, standing in for consumers such as `modules_mapping`.

## Steps to reproduce

1. Clone this repo and `cd` into it.
2. Run the two commands below (Bazel 9.3.2 is picked up from `.bazelversion`).
3. Observe that macOS passes and Linux fails.

```sh
bazel build //:all_whls --platforms=//platforms:osx_aarch64    # succeeds
bazel build //:all_whls --platforms=//platforms:linux_x86_64   # fails
```

Expected on Linux: analysis succeeds, or the macOS-only package is skipped.

Actual on Linux:

```text
configurable attribute "actual" in @@rules_python++pip+python_deps//appnope:_no_matching_repository
doesn't match this configuration: No matching wheel for current configuration's Python version and platform.
```

## Regression check

Changing `bazel_dep(name = "rules_python", version = ...)` in `MODULE.bazel` to `2.4.6` makes both commands above succeed.
So the failure is introduced between 2.4.6 and 2.5.1, consistent with the uv.lock edge-marker handling added in 2.5.0.

## Workaround

Filter the macOS-only package out of the flat list, as we do in our monorepo for `modules_mapping`:

```starlark
filegroup(
    name = "all_whls_linux",
    # appnope is macOS-only (its only edge has a darwin marker), so it has no wheel on Linux.
    srcs = [w for w in all_whl_requirements if "//appnope:" not in w],
)
```

```sh
bazel build //:all_whls_linux --platforms=//platforms:linux_x86_64   # succeeds
```

This is ugly because the exclusion list is hand-maintained.
Any other platform-exclusive package needs another entry, for example `triton` or the `nvidia_*` wheels when building for macOS.
It also means the resulting manifest has no entry for the excluded package.
