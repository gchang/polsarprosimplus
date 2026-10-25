# Changes from upstream

Upstream: https://github.com/nasa-jpl/PolSARproSimPlus, branch `main` at commit 25bcbdaef88a07778499e58a6ef601af92ed360d (2026-04-15, "added LICENSE file"). PolSARproSim+ is by Razi Ahmed and colleagues at JPL, building on Mark L. Williams' PolSARproSim (2007), with a few files from ESA's PolSARpro by Eric Pottier and Laurent Ferro-Famil. Upstream has no tags or releases.

This copy starts from upstream's file tree at 25bcbda without its git history. Nothing was left out: upstream has no `.github/` folder, and every tracked file, including `docs/`, `scripts/`, `src/attic/` and `include/attic/`, is copied as is. The simulation code, the example inputs and every license notice are untouched except for the items below.

## License

Kept exactly as upstream has it. `LICENSE.txt` is a Caltech copyright notice (2026) with BSD-style redistribution terms, followed by "Open Source License Approved by Caltech/JPL: Apache License, Version 2.0". The source files say something different: 94 of the 98 `.c` and `.h` files open with Mark L. Williams' 2007 PolSARproSim notice, which licenses them under the GNU General Public License, version 2 or (at your option) any later version. The other four (`include/graphics.h`, `include/processing.h`, `include/attic/matrix.h`, `include/attic/util.h`) carry ESA PolSARpro author headers with no license statement. Upstream does not explain how the two licenses relate. Both are left as they are here, and the three changes below are covered by whatever terms apply to the files they touch.

## 2026-10-25: meson build defines `_DEFAULT_SOURCE`

`meson.build` asks for `c_std=c11`. With glibc that is strict ISO C, which hides the POSIX random-number functions the code uses (`erand48` in `src/MonteCarlo.c`, `rand_r` in `src/PolSARproSim_Short_Vegi.c`). Older GCC only warned about the implicit declarations; GCC 14 and newer stop with an error, so `ninja` failed on current Linux compilers. macOS headers expose these functions anyway.

`meson.build` now adds `-D_DEFAULT_SOURCE` to the project's compiler arguments. The language standard stays c11. Results do not change (the machine code is identical, see Check). The Makefile does not set `-std` and is not affected.

## 2026-10-25: global mutexes and thread counter declared `extern`

Eight globals were defined, not just declared, in headers: `PolSARproSim_HHmutex`, `_HVmutex`, `_VVmutex`, `_MaxHgtmutex` and `_Statmutex` (`include/PolSARproSim.h`), `PolSARproSim_GNDmutex` (`include/PolSARproSim_Direct_Ground.h`), `PolSARproSim_Forest_Thread_Counter` (`include/PolSARproSim_Forest.h`) and `ShortVeg_mutex` (`include/PolSARproSim_Short_Vegi.h`). Every source file that includes `PolSARproSim.h` got its own copy, which only links with the old "common symbol" behavior. GCC 10 and newer default to `-fno-common`, so both the meson and the make builds stopped with "multiple definition" errors.

The headers now declare them `extern`, and each is defined once: the five image and statistics mutexes in `src/PolSARproSim.c`, the others in the source file that goes with their header. They are still zero-initialized, as before. Results do not change.

## 2026-10-25: file name buffers have room for the terminating NUL

`Create_SAR_Filenames` (`src/PolSARproSim_Procedures.c`) builds each output file name with `strcpy` and `strncat` into a buffer sized as the sum of the parts, with no byte for the terminating NUL; `main()` (`src/PolSARproSim.c`) does the same when it copies the three command-line arguments. With the README example on Linux (`pspsim run1.sar run1 out/`), `out/run1_track000_HH.bin` fills its buffer exactly, the NUL lands on the next heap block's header, and the run aborts with `malloc(): invalid next size` right after the attenuation maps. Whether it crashes depends on the lengths of the prefix and output folder and on the allocator, so it may never show on other systems.

All 15 allocations of this form get one more byte. Nothing else changes, so results do not change; runs that crashed now finish.

## 2026-10-25: environment.yml for the build

The README names the build tools (meson, ninja) and libxml2 only in passing, so there was no single list to install from. `environment.yml` lists a conda-forge build environment: a C compiler, meson, ninja, pkg-config and libxml2 with its headers. It was checked by building from a clean clone and running the example with gcc 15.3 (the file's current resolution) as well as gcc 14.4: exit 0 in 6.9 min with 182 MB peak memory, ground sigma0 identical (HH -31.08, HV -87.30, VV -26.15 dB) and forest sigma0 within 0.03 dB of the earlier runs (HH +4.20, HV -5.61, VV +1.45 dB), inside the run-to-run spread described under Check. No code changes.

## Known issues left as they are

These would change the simulation's output or statistics, so they are left for the authors to decide:

- `Normal_Distribution_r` (`src/MonteCarlo.c`, the reentrant normal generator) adds 13 uniform numbers and subtracts 6, so its draws have mean 0.5 and variance 13/12 instead of 0 and 1 (the non-reentrant `Normal_Distribution` adds 12). It drives the position and moisture changes between tracks in the temporal-decorrelation model (`src/PolSARproSim_Forest.c`, `Model_Change`), which would be biased by half a standard deviation. Single-track runs such as the example do not use it.
- The noise level (NESZ) in the parameter file is read but has no effect, because `ADD_THERMAL_NOISE` is commented out in `include/PolSARproSim_Definitions.h`. Images therefore contain no thermal noise (in the example, open-ground HV is near -87 dB rather than at the -45 dB noise floor the parameter file asks for).

Other things worth knowing when running it, also unchanged:

- The output folder must exist before the run; the program does not create it.
- `forest_file` and `allometry_file` in the parameter file are read relative to the working directory, so run from the folder that holds them (the example's `run1.sar` names bare files).
- The images are headerless complex64 binaries; their size is written only to `<prefix>.out` and the log.
- `meson.build` uses `include_directories('../include')`, which works because the README's `builddir` sits one level below the top folder.
- The README mentions a `bin/` folder that is not in the repository.

## Check

On 2026-10-25, on Linux x86-64 with conda-forge GCC 14.4.0, meson 1.12.1, ninja 1.13.2 and libxml2 2.15.4 (on conda-forge the headers and `libxml-2.0.pc` come in the separate `libxml2-devel` package), from a fresh clone of this repository and exactly as the README says:

```sh
meson setup builddir
cd builddir
ninja
```

The build succeeds with no extra options (3 s, 176 warnings, none new). Upstream's tree fails at the `erand48` error, and with that worked around, at the link. The repository has no tests. The README example, run on 2 cores from a copy of `example/` with an empty `out/` folder:

```sh
pspsim run1.sar run1 out/
```

finishes in 6 min 46 s with 182 MB peak memory and writes the four 223 by 187 complex64 images (333,608 bytes each) and the usual ancillary files. Unchanged upstream code built the same way aborts on this run.

To confirm results do not change, the same compiler built upstream's tree with only the file name fix applied, using build options instead of the first two changes (`-Dc_std=gnu11 -Dc_args=-fcommon`):

- The machine code is identical. All 43 object files disassemble the same, as does the linked program once addresses are ignored. The only difference is where the eight globals sit in memory.
- Two runs of each build agree within run-to-run variation. The program is not bit-for-bit repeatable with a fixed seed: the threads add into shared images in whatever order they finish, and probably also take numbers from the shared `rand()` stream (`drand()` in `src/MonteCarlo.c`) in that order. The attenuation maps vary from run to run for the same reason. Images from four of the runs agree with each other within 0.005 dB in scene power and 0.1% of peak amplitude. One run of the comparison build is 0.02 to 0.03 dB away from all the others, including the other run of the same build.
- The non-image outputs that do not depend on thread timing (ground and maximum height, shadow map, surface normals, line of sight) are byte-identical.
- Mean sigma0 in the forest is HH +4.23, HV -5.58, VV +1.46 dB, and on open ground HH -31.08, HV -87.30, VV -26.15 dB, the same as before the changes.
