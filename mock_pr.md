# Mock PR: Add package k4fwcore


## Description

This PR adds the `k4fwcore` package to the Spack builtin repository. k4FWCore is the core software framework for the Key4HEP project, providing the integration layer between the EDM4hep event data model and the Gaudi framework. It is used across multiple HEP experiments including FCC, ILC, CEPC, and EIC.

k4FWCore was previously only available through the third-party [key4hep-spack](https://github.com/key4hep/key4hep-spack) repository. All of its dependencies (`edm4hep`, `podio`, `gaudi`, `root`) are already in Spack builtin, making it ready for upstreaming with zero dependency blockers.

### Changes from key4hep-spack recipe

The following changes were made to bring the package into compliance with Spack's packaging guidelines:

1. **Added SPDX license header** — Required by `spack license verify` and Spack CI.

2. **Added `license("Apache-2.0")` directive** — Declares the software license per the Package Review Guide. Verified from [k4FWCore LICENSE file](https://github.com/key4hep/k4FWCore/blob/main/LICENSE).

3. **Added `maintainers()` directive** — Per the Review Guide: "If the new package does not have a maintainers directive, ask the Contributor to add one." Added relevant GitHub accounts.

4. **Removed non-portable `Ilcsoftpackage` import** — The original recipe imports `from spack.pkg.k4.key4hep_stack import Ilcsoftpackage`, which references the key4hep-spack repo namespace. This would crash in builtin where `spack.pkg.k4` does not exist. Replaced by:
   - Inheriting directly from `CMakePackage`
   - Adding `tags = ["hep", "key4hep"]` inline
   - The `url_for_version()` helper from `Ilcsoftpackage` is not needed since k4fwcore uses explicit URLs with `sha256` checksums

5. **Removed `LD_LIBRARY_PATH` manipulation** — The `setup_run_environment` method manually prepends `self.prefix.lib` to `LD_LIBRARY_PATH`. Spack handles library resolution through RPATH at build time, making this unnecessary and potentially harmful.

6. **Added `commit=` hashes for tag-only versions** — Versions `1.0pre19` and `1.0pre18` used `tag=` without `commit=`. Git tags are mutable; commit hashes ensure reproducible builds. *(In a real PR, the exact commit SHAs would be looked up from the k4FWCore GitHub repository.)*

### What the patched recipe would look like (sketch)

```python
# Copyright Spack Project Developers. See COPYRIGHT file for details.
#
# SPDX-License-Identifier: (Apache-2.0 OR MIT)

from spack.package import *


class K4fwcore(CMakePackage):
    """Core software framework for the Key4HEP project, integrating
    EDM4hep with the Gaudi framework."""

    homepage = "https://github.com/key4hep/k4FWCore"
    url = "https://github.com/key4hep/k4FWCore/archive/refs/tags/v01-01.tar.gz"
    git = "https://github.com/key4hep/k4FWCore.git"

    tags = ["hep", "key4hep"]

    maintainers("vvolkl", "jmcarcell", "tmadlener")

    license("Apache-2.0")

    # Versions listed in descending order (newest first)
    version("1.1", sha256="...")
    version("1.0", sha256="...")
    version("1.0pre19", tag="v01-00pre19", commit="<sha>")
    version("1.0pre18", tag="v01-00pre18", commit="<sha>")

    variant("cxxstd", default="17", values=("17", "20"), multi=False,
            description="C++ standard")

    depends_on("cmake", type="build")
    depends_on("edm4hep")
    depends_on("podio")
    depends_on("gaudi")
    depends_on("root")

    def cmake_args(self):
        args = [
            self.define_from_variant("CMAKE_CXX_STANDARD", "cxxstd"),
        ]
        return args
```

*Note: This is a simplified sketch to illustrate the structural changes. The actual PR would preserve all existing variants, dependencies, and version entries from the key4hep-spack recipe while applying the fixes described above.*

## Build Verification

### Platform

```
* Spack: 1.2.0.dev0 (b67fc8162421055c4c08b80a9da7f230481f0a44)
* Builtin repo: spack-packages (4da02234ab2f042bcdc6fe398cebbdc4704a9cc9)
* Python: 3.10.12
* Platform: linux-ubuntu22.04-skylake
```

### Validation Plan

The central goal of validation is to prove that the upstreamed recipe works **independently of the key4hep-spack repository** — that is, a user with only Spack's builtin repository can install k4fwcore without any additional configuration.

#### Step 0: Create a clean Spack environment

Start from a fresh Spack clone with no third-party repos registered. This ensures the test is not accidentally resolving packages from key4hep-spack:

```bash
git clone https://github.com/spack/spack.git ~/spack-test
source ~/spack-test/share/spack/setup-env.sh

# Confirm only builtin is registered
spack repo list
# Expected output: only "builtin" — no key4hep-spack
```

Copy the patched `k4fwcore/package.py` into the builtin repository:

```bash
mkdir -p $(spack repo list | grep builtin | awk '{print $2}')/packages/k4fwcore
cp patched_package.py $(spack repo list | grep builtin | awk '{print $2}')/packages/k4fwcore/package.py
```

#### Step 1: Static checks (no build required)

These catch style and metadata issues before attempting a build:

```bash
# PEP 8 and Spack style conformance
spack style --fix k4fwcore

# Package audit (checks directives, attributes, properties)
spack audit packages k4fwcore

# License header verification
spack license verify
```

All three must pass without errors.

#### Step 2: Verify the concretizer resolves all dependencies from builtin

```bash
spack spec k4fwcore
```

This is the critical independence test. If any dependency is missing from builtin, the concretizer will fail with an `UnknownPackageError`. A successful `spack spec` output means every direct and transitive dependency can be resolved without key4hep-spack. Review the output to confirm no dependency is coming from an external repo namespace.

#### Step 3: Build from source

```bash
spack install k4fwcore
```

Monitor the build log for:
- All dependencies fetched and built successfully
- No import errors (the old `from spack.pkg.k4...` import would crash here)
- No CMake configuration errors
- Successful compilation and installation

For the PR, attach the full build log or at minimum the final summary showing success.

#### Step 4: Verify the installation

```bash
# Check that the package is registered as installed
spack find k4fwcore

# Verify key installed files exist
spack find --paths k4fwcore
ls $(spack location -i k4fwcore)/lib/
```

#### Step 5: Run package tests (if defined)

```bash
spack test run k4fwcore
```

If the package defines `test()` methods, this runs them against the installed package. Test failures may indicate missing runtime dependencies or incorrect environment setup.

#### Step 6: Cross-platform verification (recommended)

The Spack contribution guide asks for testing on "at least one platform." Ideally, test on:
- Linux x86_64 (primary — this is what Spack CI runs)
- macOS (if available — catches portability issues like hardcoded Linux paths)

Include `spack debug report` output from each platform in the PR.

#### Why this proves independence

The key4hep-spack repository is never registered during this process. If any step succeeds, it means the recipe is fully self-contained within Spack builtin. Specifically:
- Step 2 proves all dependencies resolve from builtin alone
- Step 3 proves the Python recipe executes without importing from `spack.pkg.k4`
- Steps 4-5 prove the installed package is functional without `LD_LIBRARY_PATH` hacks

## Dependency Justification

All direct dependencies of k4fwcore are present in Spack builtin:

| Dependency | Builtin status | Notes |
|---|---|---|
| cmake | Present in builtin | Build-only dependency |
| edm4hep | Present in builtin | Recently upstreamed |
| podio | Present in builtin | Recently upstreamed |
| gaudi | Present in builtin | Long-standing package |
| root | Present in builtin | Long-standing package |

Zero blockers — this package is ready for upstreaming.

## Summary/Quick Checklist

- [x] Package has `maintainers()` directive
- [x] Package has `license()` directive with correct SPDX identifier
- [x] Package has proper SPDX license header
- [x] All imports are portable (no external repo references)
- [x] Versions listed in descending order
- [x] Tag-only versions have `commit=` for reproducibility
- [x] No hardcoded `LD_LIBRARY_PATH`
- [x] CMake arguments use `self.define()` / `self.define_from_variant()`
- [x] `spack debug report` output included
- [ ] Build confirmed on at least one platform *(would be done for real PR)*
