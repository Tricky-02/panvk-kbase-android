# PanVK Build Analysis: POCO X6 Pro

Status: pinned-source Android/Bionic build and static binary validation
complete. No on-device runtime or DroidDeck integration tests have been run.

## Snapshot and target

This analysis is based on the clean `main` checkout at
`5b87b74d3e2580d5816e7f2c629f6f016a462f28`. The fork's `origin` is
`Tricky-02/panvk-kbase-android`; `upstream` is
`zenithblue-oss/panvk-kbase-android`.

The repository pins Mesa commit
`5a07217f034b3e50d8c7c7794f97a2df1742613b` (`26.3.0-devel`) in
[`sources.lock`](../sources.lock). The target profile is
[`g615-v11-csf`](../profiles/g615-v11-csf.json): POCO X6 Pro (`duchamp`),
Mali-G615 MC6 (`0xb8a31030`), PAN_ARCH 11, CSF, `mali_kbase`, and the
profile's recorded Kbase UAPI 1.21. The profile also records an Android 16
device probe; those are repository-recorded results, not a fresh probe of the
user's phone.

## Build inputs and patch selection

[`fetch-mesa.sh`](../scripts/fetch-mesa.sh) reads the repository and commit
from `sources.lock`, fetches that exact commit, checks it out detached, and
removes selected generated/untracked Mesa source paths. Do not use it against
an existing `work/mesa` checkout with user data that must be retained.

[`apply-patches.sh`](../scripts/apply-patches.sh) reads the profile's
`panArch` and `frontend`. For this CSF profile it applies, in order, the
`common`, `android`, `kbase-common`, `app-loader`, `wsi`, `csf`, and
`csf-v11` families, followed by `jm-v9` and `jm-v7` for the universal ICD.
It checks each patch before applying and exits on patch drift. These scripts
operate on the Mesa tree; they are not read-only inspections.

### Patch-series provenance

`sources.lock` originally recorded
`sha256:374b7b830111a848515d3e0ec41a60b902bc57938deb420ecea7de8a03f58ecd`,
but `compute-patch-series-id.py --profile g615-v11-csf` calculated
`sha256:2837740bc363fa778ae578ae856bc5040f3ed1924e01f35d78371cd072a71f90`.
The calculator hashes the Mesa commit, normalized profile JSON, and sorted
patch/overlay files. After confirming that all 179 selected patches applied
to the pinned Mesa checkout, `sources.lock` was updated to the calculated
value. Re-running the calculator produces the value now recorded in the lock.

## Build targets

The project intentionally produces two different AArch64 driver ABIs, as
described in [`PORTABILITY.md`](PORTABILITY.md):

| Target | Intended runtime | Build path and output |
| --- | --- | --- |
| Android/Bionic | Android Vulkan loader; Android WSI/AHB and Kbase | [`build-android.sh`](../scripts/build-android.sh) cross-compiles with the Android NDK and produces `dist/android-g615-v11-csf/libvulkan_panfrost.so`. |
| Linux/glibc | ARM64 glibc runtime/PRoot consumer | [`build-glibc.sh`](../scripts/build-glibc.sh) builds the ICD and `panfrost_icd.aarch64.json` under `dist/glibc-g615-v11-csf/`. |

Do not interchange these binaries or put both ABIs in one loader directory.
The glibc ICD is not an Android/Bionic driver, and a successful glibc build
does not by itself prove access to Android's kernel GPU interface.

### Android/Bionic path

The POCO guide, [`BUILD-POCO-X6-PRO.md`](BUILD-POCO-X6-PRO.md), lists an
Android NDK and host tools as prerequisites. The build script defaults to
`/opt/android-sdk/ndk`, selects an NDK Linux x86_64 toolchain, and defaults to
Android API 29 unless `ANDROID_API` or `--api` overrides it. It builds native
Mesa host code generators, prepares X11 headers, then configures Mesa for the
PanVK Vulkan driver, Kbase, Android/X11 platforms, and no Gallium/OpenGL.
The artifact is the Bionic `libvulkan_panfrost.so`.

The guide's baseline command sequence is:

```sh
./scripts/fetch-mesa.sh
./scripts/bootstrap-host-tools.sh
./scripts/apply-patches.sh --profile g615-v11-csf
./scripts/build-android.sh --profile g615-v11-csf
./scripts/validate-binary.sh --abi android dist/android-g615-v11-csf/libvulkan_panfrost.so
```

The build script defaults to API 29, while [`RELEASE.md`](RELEASE.md) says not
to advertise API 29 until that rebuild is proven, and
[`validate-package.sh`](../scripts/validate-package.sh) rejects an Android
package with `minApi < 35` unless `minApiProven` is set. This build explicitly
used API 35; it does not establish support for API 29.

### Phase 3 build result

The build was run on Ubuntu 24.04 / WSL2 x86_64 with the profile's pinned
Mesa commit. Mesa host tools built successfully (686 Ninja tasks), then the
Android PanVK build completed (1,034 Ninja tasks) with NDK r30
(`Pkg.Revision` 30.0.16248370), Clang 21.0.0, and Android API 35. The build artifact is
`dist/android-g615-v11-csf/libvulkan_panfrost.so`, Build ID
`802fbea63406d6fd12e66ea37b86ee7f647e3f50`, SHA-256
`8b959c845f693bc22ec99887764c0d4c3ccb80840b76d896becb73429870b057`.

The build scripts expect target dependencies in `work/android-deps` and
X11/XCB headers in `work/android-deps-x11`; those were not present in the
fresh checkout. To satisfy the target `libdrm` dependency without using host
libraries, libdrm 2.4.125 was built as a static Android dependency from the
official source archive (SHA-256
`d4bae92797a50f81a93524762e0410a49cd84cfa0f997795bc0172ac8fb1d96a`). The
X11 header prefix was prepared from host development headers, including XCB
extension headers. These dependencies and build outputs remain under ignored
`work/` and `build/` directories; they are not tracked deliverables.

The target libdrm dependency was prepared with:

```sh
mkdir -p work/android-deps/lib/pkgconfig
mkdir -p work/libdrm-download
curl -L -o work/libdrm-download/libdrm-2.4.125.tar.xz \
  https://dri.freedesktop.org/libdrm/libdrm-2.4.125.tar.xz
echo "d4bae92797a50f81a93524762e0410a49cd84cfa0f997795bc0172ac8fb1d96a  work/libdrm-download/libdrm-2.4.125.tar.xz" \
  | sha256sum -c -
mkdir -p work/libdrm-src
tar -xf work/libdrm-download/libdrm-2.4.125.tar.xz \
  -C work/libdrm-src --strip-components=1
PATH="/home/tricky/.venvs/panvk-build/bin:/home/tricky/android-sdk/ndk/android-ndk-r30/toolchains/llvm/prebuilt/linux-x86_64/bin:$PATH" \
  meson setup build/libdrm-android work/libdrm-src \
  --cross-file build/android-bionic.cross.ini \
  -Dprefix="$PWD/work/android-deps" -Ddefault_library=static \
  -Dintel=disabled -Dradeon=disabled -Damdgpu=disabled \
  -Dnouveau=disabled -Dvmwgfx=disabled -Domap=disabled \
  -Dexynos=disabled -Dfreedreno=disabled -Dtegra=disabled \
  -Dvc4=disabled -Detnaviv=disabled -Dman-pages=disabled \
  -Dtests=false -Dinstall-test-programs=false -Dudev=false -Dvalgrind=disabled
PATH="/home/tricky/.venvs/panvk-build/bin:/home/tricky/android-sdk/ndk/android-ndk-r30/toolchains/llvm/prebuilt/linux-x86_64/bin:$PATH" \
  ninja -C build/libdrm-android
PATH="/home/tricky/.venvs/panvk-build/bin:/home/tricky/android-sdk/ndk/android-ndk-r30/toolchains/llvm/prebuilt/linux-x86_64/bin:$PATH" \
  ninja -C build/libdrm-android install
./scripts/prepare-x11-headers.sh work/android-deps-x11
```

`build-android.sh` only adds the target dependency and X11 pkg-config
directories to its cross file when `work/android-deps/lib/pkgconfig` already
exists. The project does not currently provide a script to build or pin this
Android libdrm dependency; the source archive digest above records the exact
input used for this result.

Commands used after installing prerequisites:

```sh
./scripts/fetch-mesa.sh
./scripts/apply-patches.sh --profile g615-v11-csf --mesa work/mesa
PATH=/home/tricky/.venvs/panvk-build/bin:$PATH \
  taskset -c 0-3 ./scripts/bootstrap-host-tools.sh \
  --mesa work/mesa --out build/host-toolsmesa
ANDROID_NDK_ROOT=/home/tricky/android-sdk/ndk \
ANDROID_API=35 \
HOST_TOOLS="$PWD/build/host-toolsmesa/bin" \
PATH="/home/tricky/.venvs/panvk-build/bin:$PATH" \
  taskset -c 0-3 ./scripts/build-android.sh \
  --profile g615-v11-csf --api 35 --ndk /home/tricky/android-sdk/ndk
./scripts/validate-binary.sh --abi android \
  dist/android-g615-v11-csf/libvulkan_panfrost.so
```

The static validator passed. The artifact is AArch64 ELF and exports
`vk_icdGetInstanceProcAddr` and
`vk_icdNegotiateLoaderICDInterfaceVersion`; its recorded dependencies are
Android system libraries (`liblog`, `libnativewindow`, `libsync`, `libm`,
`libz`, `libdl`, `libc`), with no glibc or X11 shared-library dependency.
The build emitted one C23-extension warning in `csf_panvk_vX_gpu_queue.c`;
it did not prevent the build or static validation.

### glibc path

On an AArch64 Linux host, `build-glibc.sh` configures Mesa natively and emits
the shared library plus the ICD JSON manifest. On a non-AArch64 host, the
script instead requires an ADB serial and dispatches the build into a
device-side glibc chroot through its configurable wrapper (by default
`/data/local/tmp/nativecode_chroot.sh`). Its source stage is a deterministic
archive of the patched Mesa tree, and the remote build requires the
corresponding ARM64 glibc/PRoot environment. The script reconstructs its
generated source tree under `build/`; check for existing user data there
before running it.

The WSL environment is Ubuntu 24.04 on WSL2, host architecture `x86_64`.
NDK r30 is installed under `/home/tricky/android-sdk/ndk/android-ndk-r30`.
The ADB client is installed, but `adb devices -l` reports no attached or
authorized device. The documented glibc build route (which dispatches to an
ARM64 device chroot from this host) was therefore not run. A native glibc
build is not directly available on this x86_64 host.

## Validation and evidence

[`validate-binary.sh`](../scripts/validate-binary.sh) performs static checks:
ELF details, dependencies and ICD exports, plus ABI-specific checks. For
Android it checks for ICD exports and rejects glibc `libc.so.6` and X11
dependencies; for glibc it checks for `libc.so.6`. This is not a runtime
driver-load or rendering test.

The repository's [`tests/README.md`](../tests/README.md) describes a stronger
sequence: app-UID Kbase probe, Vulkan loader/instance/device initialization,
compute, offscreen rendering, AHB, Android surface/swapchain, synchronization,
and bounded soak tests. [`validate-package.sh`](../scripts/validate-package.sh)
also enforces provenance, package ABI/profile, and recorded validation gates.
Passing static binary validation must not be presented as passing these
runtime gates.

No Android device tests were run. ADB was not installed or available from
Windows and no device is attached to WSL, so driver loading, Vulkan
enumeration, rendering, presentation, and game compatibility remain
unverified by this build session. `tests/validate-package-selfcheck.py` passed;
it tests package validation rules using synthetic fixtures and is not a
runtime driver test.

The project README records prior G615 device results, including Android
loader/WSI, glibc ICD, feature workloads, and DXVK Native D3D11/D3D9
workloads. These are upstream repository claims/evidence, not tests rerun in
this phase. The same README reports that ordinary Proton 11 i686/WoW64 surface
creation is blocked before the Android driver is called and that D3D12 remains
blocked by missing requirements. See [`README.md`](../README.md), especially
the tested-device and DXVK/vkd3d sections.

## Constraints and next checks

- The upstream project says its canonical target is the vendor
  `mali_kbase` interface (`/dev/mali0`); this is distinct from upstream Mesa's
  Panthor DRM backend. See [`MALI-GPU-ARCHITECTURES.md`](MALI-GPU-ARCHITECTURES.md)
  and [`KBASE-PROFILES.md`](KBASE-PROFILES.md).
- The profile records app-UID access on its tested device, but this must be
  rechecked on the actual stock phone. The build guide's development loop
  mentions a rooted phone; do not turn root, `/dev/mali0` permission changes,
  or system-partition changes into assumptions for this project.
- Before another build, inspect the ignored `work/` and `build/` directories;
  the Mesa fetch script resets the pinned source checkout and the glibc
  script reconstructs its source tree.
- Keep driver validation distinct from DroidDeck runtime/compositor
  integration. The latter is not demonstrated by building either ABI.
- Shader-cache/precompilation research is out of scope until the selected
  driver/runtime path and its frame-presentation behavior have been
  independently validated.
