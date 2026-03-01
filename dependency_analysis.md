# Dependency Graph & Blocker Analysis

## Concretized Spec

The following spec was generated using `spack spec k4fwcore`, showing the full concretized dependency tree with exact versions, variants, compilers, and platform:

```
$ spack spec k4fwcore

k4fwcore@1.5~ipo build_system=cmake build_type=Release generator=make
  platform=linux os=ubuntu22.04 target=skylake %cxx=gcc@11.4.0
    ^edm4hep@1.0~ipo+json cxxstd=20 %cxx=gcc@11.4.0
    ^gaudi@40.2~aida~cuda~docs~examples~heppdt~ipo~xercesc cxxstd=20 %cxx=gcc@11.4.0
        ^boost@1.89.0  ^clhep@2.4.7.2  ^fmt@10.2.1  ^intel-tbb@2022.3.0
        ^range-v3@0.12.0  ^py-six@1.17.0  ^py-networkx@2.7.1  ...
    ^podio@1.7~datasource~ipo~rntuple~sio cxxstd=20 %c,cxx=gcc@11.4.0
    ^root@6.38.02+python+roofit+tbb+x+xml cxxstd=20 %c,cxx=gcc@11.4.0
        ^davix@0.8.10  ^mesa@25.0.5  ^openssl@3.6.1  ^python@3.12.12  ...
    ^cmake@3.31.11  ^gmake@4.4.1  ^gcc-runtime@11.4.0
```

*(Output abbreviated — full spec resolves ~120 transitive dependencies, all from Spack builtin.)*


## Graph Generation

Dependency graphs were generated using the following:

```bash
spack graph --dot k4fwcore > k4fwcore_graph.dot
dot -Tpng k4fwcore_graph.dot -o dependency_analysis.png
```

The resulting graph (see `dependency_analysis.png`) shows the full transitive dependency tree. k4fwcore pulls in a large number of indirect dependencies through `root`, `gaudi`, and `geant4`, which is typical for HEP software stacks. However, the graph is quite complex and it is hard to make out individual connections visually.


## k4fwcore: Dependency Blocker Analysis: 0 Blockers

Direct dependencies from `spack dependencies k4fwcore`, cross-referenced with `grep "depends_on" key4hep-spack/packages/k4fwcore/package.py`:

| Dependency | Declared in `package.py`? | In Spack builtin? |
|---|---|---|
| gaudi | `depends_on("gaudi")` | Yes |
| root | `depends_on("root")` | Yes |
| podio | `depends_on("podio")` | Yes |
| edm4hep | `depends_on("edm4hep")` | Yes |
| cmake | build system requirement | Yes |
| gcc, llvm, fj, acfl, aocc, cce, nvhpc, xl | compilers (not in `depends_on`) | N/A |
| gmake, ninja | build tools | Yes |

**Result: Zero blockers.** All declared dependencies are in Spack builtin. k4fwcore can be upstreamed immediately.

## k4simgeant4: Dependency Blocker Analysis: 3 Blockers

Direct dependencies from `spack dependencies k4simgeant4`, cross-referenced with `grep "depends_on" key4hep-spack/packages/k4simgeant4/package.py`:

| Dependency | Declared in `package.py`? | In Spack builtin? | Blocker? |
|---|---|---|---|
| clhep | `depends_on("clhep")` | Yes | — |
| dd4hep | `depends_on("dd4hep")` | Yes | — |
| geant4 | `depends_on("geant4")` | Yes | — |
| edm4hep | `depends_on("edm4hep")` | Yes | — |
| g4ensdfstate | `depends_on("g4ensdfstate")` | Yes | — |
| root | `depends_on("root")` | Yes | — |
| cmake | build system requirement | Yes | — |
| **k4fwcore** | `depends_on("k4fwcore")` | **No**, key4hep only | **BLOCKER** |
| **k4gen** | `depends_on("k4gen", type="test")` | **No**, key4hep only | **BLOCKER** (test dep) |
| **fccdetectors** | `depends_on("fccdetectors", type="test")` | **No**, key4hep only | **BLOCKER** (test dep) |

**Result: 3 blockers.** k4simgeant4 cannot be upstreamed until k4fwcore, k4gen, and fccdetectors are upstreamed first.

## ilcutil: Dependency Blocker Analysis: 0 Blockers

Direct dependencies from `spack dependencies ilcutil`, cross-referenced with `grep "depends_on" key4hep-spack/packages/ilcutil/package.py`:

| Dependency | Declared in `package.py`? | In Spack builtin? |
|---|---|---|
| doxygen | `depends_on("doxygen", when="+doc")` | Yes |
| cmake | build system requirement | Yes |

ilcutil is a leaf utility package with no key4hep-only dependencies.

**Result: Zero blockers.** ilcutil can be upstreamed immediately.

---

## Upstreaming Order

This analysis demonstrates why dependency-aware ordering is essential. The required sequence for these three packages would be:

```
Wave 1 (leaf nodes, zero blockers):
  ├── ilcutil
  └── k4fwcore

Wave 2 (unblocked after Wave 1):
  └── k4gen (depends on k4fwcore)

Wave 3 (unblocked after Wave 2):
  └── k4simgeant4 (depends on k4fwcore + k4gen)
```

Note: `fccdetectors` (a test-only dependency of k4simgeant4) would also need to be upstreamed before or alongside Wave 3, or the test dependency could be made conditional/optional in the upstreamed recipe.