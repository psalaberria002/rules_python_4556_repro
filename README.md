# Reproduction for rules_python#4556

Reproduces <https://github.com/bazel-contrib/rules_python/issues/4556>.
`modules_mapping` from `rules_python_gazelle_plugin` (the input to `gazelle_python_manifest`) fails analysis on a platform where a platform-exclusive package is unreachable.
This is with rules_python 2.5.1 and `pip.parse(uv_lock = ...)`.
The cause is that `all_whl_requirements`, which `modules_mapping` takes as `wheels`, is a flat list of every package regardless of platform.

## Setup

- `pyproject.toml` depends on `ipykernel` and locks for Linux and macOS in one universal `uv.lock`.
- The lock contains the edge `{ name = "appnope", marker = "sys_platform != 'linux'" }`, so `appnope` is macOS-only.
- `MODULE.bazel` calls `pip.parse` with `target_platforms = ["linux_aarch64", "linux_x86_64", "osx_aarch64"]`.
- `//:modules_map` is a `modules_mapping` with `wheels = all_whl_requirements`, as in `examples/build_file_generation`.

## Steps to reproduce

1. Clone this repo and `cd` into it.
2. Run the two commands below (Bazel 9.3.2 is picked up from `.bazelversion`).
3. Observe that macOS passes and Linux fails.

```sh
bazel build //:modules_map --platforms=//platforms:osx_aarch64    # succeeds
bazel build //:modules_map --platforms=//platforms:linux_x86_64   # fails
```

Expected on Linux: analysis succeeds, or the macOS-only package is skipped.

Actual on Linux:

```text
configurable attribute "actual" in @@rules_python++pip+python_deps//appnope:_no_matching_repository
doesn't match this configuration: No matching wheel for current configuration's Python version and platform.
```

## Regression check

Changing the `rules_python` and `rules_python_gazelle_plugin` versions in `MODULE.bazel` to `2.4.6` makes both commands above succeed.
So the failure is introduced between 2.4.6 and 2.5.1, consistent with the uv.lock edge-marker handling added in 2.5.0.

## Workaround

Filter the macOS-only package out of the flat list, as we do in our monorepo for `modules_mapping`:

```starlark
modules_mapping(
    name = "modules_map_workaround",
    # appnope is macOS-only (its only edge has a darwin marker), so it has no wheel on Linux.
    wheels = [w for w in all_whl_requirements if "//appnope:" not in w],
)
```

```sh
bazel build //:modules_map_workaround --platforms=//platforms:linux_x86_64   # succeeds
```

This is ugly because the exclusion list is hand-maintained.
Any other platform-exclusive package needs another entry, for example `triton` or the `nvidia_*` wheels when building for macOS.
It also means the resulting mapping has no entry for the excluded package, so gazelle cannot resolve imports of it.
