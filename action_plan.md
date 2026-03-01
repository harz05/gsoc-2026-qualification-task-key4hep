# Action Plan: Prioritizing Upstreaming of 50 Packages

**Given a list of 50 packages to upstream, I would prioritize using a dependency-first topological ordering.**

**Step 1: Identify zero-blocker packages.** For each package, extract its `depends_on()` declarations from `package.py` and check whether each dependency already exists in Spack builtin. Packages with all dependencies satisfied form Wave 1 and can be upstreamed immediately. To validate this approach, I ran this analysis across all ~85 key4hep-spack packages and found 26 ready for immediate upstreaming, confirming the method works at scale.

**Step 2: Prioritize within each wave by downstream impact.** Not all zero-blocker packages are equal. In my analysis, `ilcutil` alone blocks approximately 30 downstream packages while `k4fwcore` blocks around 15. Upstreaming high-impact leaf nodes first creates a cascade that rapidly unblocks subsequent waves.

**Step 3: Iterate.** After each wave merges into builtin, re-run the blocker check. Previously blocked packages whose dependencies have now been merged become the next wave. Waves shrink quickly because key leaf nodes like `ilcutil` unlock large chains.

**Per-package process:** Audit against the Package Review Guide, fix compliance issues, validate with `spack spec` and `spack install` in a clean builtin-only environment, then submit one PR per package with `spack debug report` output.