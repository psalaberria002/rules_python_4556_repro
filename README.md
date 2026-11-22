# Reproduction for rules_python#4556

Reproduces <https://github.com/bazel-contrib/rules_python/issues/4556>.
Generating `gazelle_python.yaml` with `gazelle_python_manifest` fails when the `modules_mapping` is built for a platform where a platform-exclusive package is unreachable.
This is with rules_python 2.5.1 and `pip.parse(uv_lock = ...)`.
The cause is that `all_whl_requirements`, which `modules_mapping` takes as `wheels`, is a flat list of every package regardless of platform.

## Setup

This mirrors how we use rules_python and gazelle in our monorepo.

- `pyproject.toml` depends on `ipykernel` and locks for Linux and macOS in one universal `uv.lock`.
- The lock contains the edge `{ name = "appnope", marker = "sys_platform != 'linux'" }`, so `appnope` is macOS-only.
- `MODULE.bazel` calls `pip.parse` with `target_platforms = ["linux_aarch64", "linux_x86_64", "osx_aarch64"]`.
- `//:modules_map` is a `modules_mapping` with `wheels = all_whl_requirements`, as in `examples/build_file_generation`.
- `//:modules_map_linux` builds it with a transition to Linux (`build/transitions.bzl`), so the manifest is identical on every host.
- `//:gazelle_python` is the `gazelle_python_manifest` that writes `gazelle_python.yaml` from that mapping.
- `src/main.py` imports `ipykernel`, so gazelle has something to resolve.

## Steps to reproduce

1. Clone this repo and `cd` into it (Bazel 9.3.2 is picked up from `.bazelversion`).
2. Generate the manifest, on any host:

   ```sh
   bazel run //:gazelle_python.update
   ```

3. Observe that it fails, even on macOS, because the transition builds the mapping for Linux.

Expected: the manifest is generated, or the macOS-only package is skipped.

Actual:

```text
configurable attribute "actual" in @@rules_python++pip+python_deps//appnope:_no_matching_repository
doesn't match this configuration: No matching wheel for current configuration's Python version and platform.
```

Without the transition the same failure shows up for any Linux build of the mapping:

```sh
bazel build //:modules_map --platforms=//platforms:linux_x86_64   # fails
bazel build //:modules_map --platforms=//platforms:osx_aarch64    # succeeds
```

## Regression check

Changing the `rules_python` and `rules_python_gazelle_plugin` versions in `MODULE.bazel` to `2.4.6` makes `bazel run //:gazelle_python.update` succeed.
The generated manifest then contains `appnope`, because on 2.4.6 every package had a wheel on every target platform.
So the failure is introduced between 2.4.6 and 2.5.1, consistent with the uv.lock edge-marker handling added in 2.5.0.

## Workaround

Filter the macOS-only package out of the flat list, as we do in our monorepo:

```starlark
modules_mapping(
    name = "modules_map_workaround",
    # appnope is macOS-only (its only edge has a darwin marker), so it has no wheel on Linux.
    wheels = [w for w in all_whl_requirements if "//appnope:" not in w],
)
```

`//:gazelle_python_workaround` is the manifest built from it:

```sh
bazel run //:gazelle_python_workaround.update   # writes gazelle_python.yaml
bazel run //:gazelle                            # generates src/BUILD.bazel, resolving ipykernel
```

This is ugly because the exclusion list is hand-maintained.
Any other platform-exclusive package needs another entry, for example `triton` or the `nvidia_*` wheels when the mapping is built for macOS.
It also means the manifest has no entry for the excluded package, so gazelle cannot resolve imports of it.
