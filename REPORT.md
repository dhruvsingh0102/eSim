# eSim 2.5 installation on Ubuntu 25.04

Screening Task 4 for the FOSSEE Winter Internship 2026. This report is in progress.

## Baselines

- Released package: FOSSEE's [eSim 2.5 Linux ZIP](https://esim.fossee.in/downloads), SHA-256 `b9860ae26f026930bae589520761f4a57b85d593a2e2cd1ac83d5e5db2f281ed` verified locally.
- Packaging branch: [FOSSEE/eSim at a77d6b9](https://github.com/FOSSEE/eSim/tree/a77d6b904c7bab275d819cf4d8cbb594802fe16d). The branch is named `installers`.
- Guest verification target: Ubuntu 25.04 amd64, 4 GiB RAM, two vCPUs and a 40 GiB dynamically allocated disk. The host is an Apple M4 Mac. The planned UTM guest uses x86-64 emulation.

## Issue log

| Step | Observed error | Cause | Action and verification |
| --- | --- | --- | --- |
| Start the upstream branch installer | `bash -n Ubuntu/install-eSim.sh` exits 2 with `syntax error near unexpected token '}'` at line 70. | The dispatcher opens an `if` around the child script call. An unrelated installer body follows the call and leaves the dispatcher's `if` unclosed. | Restored the dispatcher from upstream commit `f4cede1`, including Ubuntu 25.04 routing, the missing-script error and status propagation. Bash syntax checks and isolated dispatcher tests pass. |
| Start the released 2.5 installer with Ubuntu 25.04 version detection | The released wrapper reports `Unsupported Ubuntu version: 25.04` in an isolated test. | The ZIP's wrapper lacks a 25.04 dispatch case, and the ZIP lacks `install-eSim-25.04.sh`. | The repaired branch installer provides both. Run with the matching 2.5 application and bundled assets in the Ubuntu guest. |

## Reproduce

From the packaging checkout, run `bash -n Ubuntu/install-eSim.sh`. To compare the original wrapper, run `git show a77d6b9:Ubuntu/install-eSim.sh | bash -n`.

In the guest, extract the verified ZIP into one directory for the released baseline and a second directory for the branch installer. In the second copy, replace `install-eSim.sh` and `install-eSim-scripts/` with the files under the packaging branch's `Ubuntu/` directory. The installer uses the current working directory to resolve application sources, libraries and `nghdl.zip`.

Run this from the root of each copy in Bash and use distinct log names for the two runs.

```bash
set -o pipefail
bash ./install-eSim.sh --install 2>&1 | tee ../run1.log
printf 'Pipeline exit status: %s\n' "$?"
```

## Verification so far

Bash parsed both the repaired dispatcher and the Ubuntu 25.04 child script. Tests using simulated Ubuntu versions and stub child scripts confirmed the expected install and uninstall routing for 22.04.4, 23.04, 24.04 and 25.04. They also covered invalid arguments, a missing child script and propagation of a nonzero child exit status.

The full Ubuntu installation and GUI launch are pending. Dependency, NGHDL and GUI results will be added after guest testing.
