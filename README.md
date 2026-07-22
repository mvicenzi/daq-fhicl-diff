# daq-fhicl-diff

Compare two artdaq/FHiCL configurations parameter by parameter.

A plain `diff` between two exported configurations is close to useless: the files are formatted
inconsistently, most of what changes is per-instance noise, and a `.fcl` file is also not
self-contained. For example, `EventBuilder1.fcl` is three lines of overrides on top 
of `EventBuilder_standard.fcl`; diffing the file text tells you nothing about 
the ~750 parameters the DAQ actually sees.

This tool parses each configuration with [`fhicl-py`][fhiclpy], so `#include` directives are
resolved and every process configuration is compared as the fully expanded parameter set. It then
reports differences layered by where the value was set, so one change in a shared
`_standard.fcl` is reported once rather than once per board.

[fhiclpy]: https://github.com/art-framework-suite/fhicl-py

## Usage

```
daqconf-diff [--label-a NAME] [--label-b NAME] [-o FILE] <configA> <configB>
```

Pass the top-level configuration directories (the ones holding `schema.fcl`/`hashes.fcl`), or
just a flat pile of `.fcl` files. The `.fcl` files are found recursively, so both layouts work:

```
Laser_MINBIAS_Run4_00002/          12814/
├── schema.fcl                     ├── icaruspmteebot01.fcl
├── hashes.fcl                     ├── icaruspmteebot02.fcl
└── Laser_MINBIAS_Run4_/           └── ...            (flat, no includes)
    ├── EventBuilder1.fcl
    └── ...
```

The inner directory is named after the configuration, so it differs between the two sides being
compared; files are therefore matched by basename, not by relative path.

`--label-a` / `--label-b` override the displayed names, which is worth doing when both directories
are named the same thing (two `conftool.py exportConfiguration` dumps, say).

```sh
daqconf-diff Physics_General_..._00015 Laser_MINBIAS_Run4_00002
daqconf-diff /tmp/cfg_old /tmp/cfg_new --label-a "run 12811" --label-b "run 12814"
```

By default everything goes to stdout; pass `-o FILE` (`--output`) to write the report to a file
instead. Files with no differences are never printed; two identical configurations print
`No differences.`

```sh
daqconf-diff /tmp/cfg_old /tmp/cfg_new -o run12814_diff.txt
```

## Grouping

`schema.fcl` is deliberately not used for grouping, though it is still compared as metadata if present.
Entities are grouped by what the files themselves say:

1. **By shared include.** Everything including `pmt_standard.fcl` forms one group, named after that
   base. This should reproduce the collections in `schema.fcl` exactly.
2. **By parameter-set structure**, for flat configurations with no includes at all (run records). Boards of the
   same type have identical key sets; the group is named after the longest common filename prefix.

## Requirements

Python3 and the `fhicl` extension module from `fhicl-py`. No third-party Python packages.

`fhicl.so` is located automatically, trying in order:

1. `import fhicl` if already on `sys.path` (e.g. after `spack load py-fhicl-py`).
2. `$PY_FHICL_PY_LIB`, if set as the directory containing `fhicl.so`.
3. `spack location -i py-fhicl-py`.
4. `/daq/software/spack_packages/py-fhicl-py/*/*/lib/fhicl.so`, newest first.

On the ICARUS DAQ machines this means the tool runs with no setup at all: `fhicl.so` is built for
Python 3.9, which is also the system Python, so step 4 finds it and no `spack load` is needed.
The tool prints every location it tried when the import fails.