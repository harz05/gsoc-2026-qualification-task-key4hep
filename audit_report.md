# Upstream Readiness Audit Report by Harsh

## Environment

* **Spack:** 1.2.0.dev0 (b67fc8162421055c4c08b80a9da7f230481f0a44)
* **Builtin repo:** spack-packages (4da02234ab2f042bcdc6fe398cebbdc4704a9cc9)
* **key4hep-spack:** main branch (cloned 2025-03-01)
* **Python:** 3.10.12
* **Platform:** linux-ubuntu22.04-skylake

## Methodology

To identify the delta between key4hep-spack and Spack builtin, the package directories were compared:

```bash
comm -23 \
  <(ls key4hep-spack/packages/ | sort) \
  <(ls spack-packages/repos/spack_repo/builtin/packages/ | sort)
```

This yielded approximately 85 packages present in key4hep-spack but absent from the Spack builtin repository. I selected 3 packages for detailed audit at different layers of the dependency hierarchy:

1. **ilcutil** — A leaf utility library with no key4hep-only dependencies
2. **k4fwcore** — The core framework component connecting EDM4hep to Gaudi
3. **k4simgeant4** — An application-level package for Geant4 simulation in Key4HEP

This selection demonstrates the upstreaming challenge at every level: `ilcutil` is a leaf utility with no key4hep-only dependencies, `k4fwcore` sits at the framework layer with all its dependencies (`edm4hep`, `podio`, `gaudi`, `root`) already present in Spack builtin, and `k4simgeant4` is blocked because it depends on `k4fwcore`, `k4gen`, and `fccdetectors` — none of which are in builtin yet. Both `ilcutil` and `k4fwcore` have zero blockers and can be upstreamed immediately, while `k4simgeant4` must wait.

---

## Automated Audit

```
$ spack audit packages k4fwcore k4simgeant4 ilcutil
PKG-DIRECTIVES: passed
PKG-ATTRIBUTES: passed
PKG-PROPERTIES: passed
```

No automated issues were detected. The manual review below identifies deviations from Spack's official packaging guidelines that `spack audit` does not currently check.

---

## Package 1: k4fwcore

**Source:** `key4hep-spack/packages/k4fwcore/package.py`

### Manual Findings

**1. Missing SPDX license header**

The file has no copyright or license header at all. According to Spack's packaging guide requires every package file to begin with:

```python
# Copyright Spack Project Developers. See COPYRIGHT file for details.
#
# SPDX-License-Identifier: (Apache-2.0 OR MIT)
```

Reference: Every builtin package (e.g., `hepmc3`) includes this header. The Spack CI enforces this via `spack license verify`.

**2. Missing `license()` directive**

The package does not declare the license of the software being packaged. As per the Package Review Guide, reviewers are instructed to ask contributors to add a `license()` directive for new packages. k4FWCore is licensed under Apache-2.0, so the package should include:

```python
license("Apache-2.0")
```

**3. Missing `maintainers()` directive**

No `maintainers()` directive is present. The Package Review Guide states that "If the new package does not have a maintainers directive, ask the Contributor to add one." This is critical for notification of PRs affecting the package.

**4. Non-portable import from key4hep-spack**

The package imports `Ilcsoftpackage` from the key4hep repo:

```python
from spack.pkg.k4.key4hep_stack import Ilcsoftpackage
```

This import will fail in Spack builtin because `spack.pkg.k4` does not exist there. Investigation shows `Ilcsoftpackage` provides:
- `tags = ["hep", "key4hep"]` (from parent class `Key4hepPackage`)
- A custom `url_for_version()` method that converts version numbers to ILC-style URLs (e.g., `1.5` → `v01-05`)

For upstreaming, this must be replaced by adding `tags` directly to the class and either inlining the URL conversion logic or relying on explicit version URLs (which k4fwcore already provides via `sha256` checksums).

**5. `LD_LIBRARY_PATH` manipulation in `setup_run_environment`**

```python
env.prepend_path("LD_LIBRARY_PATH", self.prefix.lib)
```

Spack uses RPATH to embed library paths at build time, making manual `LD_LIBRARY_PATH` manipulation unnecessary and potentially harmful (it can cause conflicts between packages). This should be removed or replaced with proper RPATH handling.

**6. Tag-only versions without commit hashes**

```python
version("1.0pre19", tag="v01-00pre19")
version("1.0pre18", tag="v01-00pre18")
```

Git tags can be moved or deleted. Spack requires a `commit=` argument alongside `tag=` for reproducibility. These should either be given commit hashes or removed if they are no longer relevant.

---

## Package 2: k4simgeant4

**Source:** `key4hep-spack/packages/k4simgeant4/package.py`

### Manual Findings

**1. Missing SPDX license header**

Same issue as k4fwcore — no copyright or license header present.

**2. Missing `license()` directive**

The software's license is not declared. k4SimGeant4 is licensed under Apache-2.0.

**3. Non-portable import from key4hep-spack**

```python
from spack.pkg.k4.key4hep_stack import Key4hepPackage
```

Same portability issue as k4fwcore. `Key4hepPackage` only provides `tags = ["hep", "key4hep"]`, which is trivial to inline.

**4. Six tag-only versions without commit hashes**

```python
version("0.1.0pre15", tag="v0.1.0pre15")
version("0.1.0pre14", tag="v0.1.0pre14")
version("0.1.0pre13", tag="v0.1.0pre13")
version("0.1.0pre12", tag="v0.1.0pre12")
version("0.1.0pre11", tag="v0.1.0pre11")
version("0.1.0pre10", tag="v0.1.0pre10")
version("0.1.0pre09", tag="v0.1.0pre09")
version("0.1.0pre08", tag="v0.1.0pre08")
```

Eight versions lack `commit=` hashes, violating reproducibility requirements. Only the two most recent versions (pre16, pre17) have `sha256` checksums.

**5. Inconsistent CMake argument style**

```python
f"-DCMAKE_CXX_STANDARD={self.spec['root'].variants['cxxstd'].value}"
```

This uses an f-string instead of the idiomatic `self.define()` method used elsewhere in the same function and throughout Spack's builtin packages. Should be:

```python
self.define("CMAKE_CXX_STANDARD", self.spec["root"].variants["cxxstd"].value)
```

**6. `LD_LIBRARY_PATH` manipulation in multiple methods**

Both `setup_run_environment` and `setup_dependent_build_environment` manually prepend to `LD_LIBRARY_PATH`. Same concern as k4fwcore — Spack uses RPATH and this is discouraged.

---

## Package 3: ilcutil

**Source:** `key4hep-spack/packages/ilcutil/package.py`

### Manual Findings

**1. Missing `license()` directive**

The software's license is not declared. ilcutil is licensed under GPL-3.0-only.

**2. Outdated license header**

The file has a license header, but it uses the old format:

```python
# Copyright 2013-2020 Lawrence Livermore National Security, LLC and other
# Spack Project Developers. See the top-level COPYRIGHT file for details.
```

This should be updated to the current Spack standard:

```python
# Copyright Spack Project Developers. See COPYRIGHT file for details.
```


**3. Non-portable import from key4hep-spack**

```python
from spack.pkg.k4.key4hep_stack import Ilcsoftpackage
```

Same portability issue. `Ilcsoftpackage` provides the ILC URL conversion logic, which ilcutil actively uses (its versions follow the `v01-06` pattern). For upstreaming, the `url_for_version()` method would need to be inlined into the package class.