# daqconf-diff

Compare two artdaq/FHiCL configurations parameter by parameter.

A plain `diff` between two exported configurations is close to useless: the files are formatted
inconsistently, most of what changes is per-instance noise, and — worst — a `.fcl` file is not
self-contained. `EventBuilder1.fcl` is three lines of overrides on top of `EventBuilder_standard.fcl`;
diffing the file text tells you nothing about the ~750 parameters the DAQ actually sees.

This tool parses each configuration with [`py-fhicl-py`][fhiclpy], so `#include` directives are
resolved and every process configuration is compared as the fully expanded parameter set. It then
reports differences **layered by where the value was set**, so one change in a shared
`_standard.fcl` is reported once rather than once per board.

For scale: two 351-file configurations differing by a single filter swap produce a **21-line**
report. The line-oriented script this replaces produced 80 KB for a comparable pair.

[fhiclpy]: https://github.com/art-framework-suite/fhicl-py

## Usage

```
daqconf-diff [--label-a NAME] [--label-b NAME] <configA> <configB>
```

Pass the **top-level configuration directories** — the ones holding `schema.fcl`/`hashes.fcl`, or
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
compared; files are therefore matched by **basename**, not by relative path.

`--label-a` / `--label-b` override the displayed names, which is worth doing when both directories
are named the same thing (two `conftool.py exportConfiguration` dumps, say).

```sh
daqconf-diff Physics_General_..._00015 Laser_MINBIAS_Run4_00002
daqconf-diff /tmp/cfg_old /tmp/cfg_new --label-a "run 12811" --label-b "run 12814"
```

Everything goes to stdout. Files with no differences are never printed; two identical
configurations print `No differences.`

## Reading the report

```
========================================================================
  A = Physics_General_..._newpeds_win_10s_00015
      /home/nfs/mvicenzi/comparing/Physics_General_..._00015
  B = Laser_MINBIAS_Run4_00002
      /home/nfs/mvicenzi/comparing/Laser_MINBIAS_Run4_00002

  All differences read  A -> B.
  [A only] = present in A, absent from B.
========================================================================

pmt -- 24 entities

 [base] pmt_standard.fcl   (applies to 24 entities unless overridden)
    [A only] daq.fragment_receiver.MajorityLevel = 0
    [B only] daq.fragment_receiver.OutputClk = False
    daq.fragment_receiver.AdcCalibration  False -> True

 [local overrides]
    daq.fragment_receiver.AdcCalibration
       [B only] icaruspmtwwtop02 = False   (A inherits from base)        ! masks base
    daq.fragment_receiver.channelPedestal0
       icaruspmteebot01   7379 -> 6940
       icaruspmteebot02   7192 -> 7199

    pmt: 1 changed, 1 A-only, 1 B-only across 24 entities
```

**Everything reads `A -> B`**, in the order given on the command line, and the banner repeats both
paths so the orientation is never in doubt.

**`[base]`** is the diff of a shared include, reported **once** for every entity that includes it.
Without this layer, one changed value in `pmt_standard.fcl` would appear 24 times.

**`[local overrides]`** covers parameters set in the individual file rather than inherited. Grouping
is **by parameter first, value second**, so an outlier board appears directly beneath the same key
as everything else rather than in a detached group. File lists collapse numeric runs
(`EventBuilder1-12`).

**`! masks base`** marks a local override of a key whose base also changed — the case where the
`[base]` line above does *not* reach that entity. In the example, `pmt_standard.fcl` flipped
`AdcCalibration` to `True`, but `icaruspmtwwtop02` pins it to `False`, so 23 boards changed and one
did not. These are reported even when the effective values on both sides agree, because that
agreement is exactly what the local pin is producing.

**`[A only]` / `[B only]`** mark a parameter present on one side only — as important as a changed
value, since it means a setting was dropped or introduced. A missing parameter is never rendered as
a value diff against a placeholder.

Wholly one-sided **subtrees** collapse to one line
(`[A only] subtree bnbfilter (5 keys: gate_type, module_type, ...)`). A *partially* present subtree
is never collapsed, so a single dropped parameter cannot hide inside a group.

Closing sections list whole files present in only one configuration (`Only in A`), differences in
`flags.fcl` / `schema.fcl`, and any file that failed to parse.

## Grouping

Entities are grouped by what the files themselves say, not by an external manifest:

1. **By shared include.** Everything including `pmt_standard.fcl` forms one group, named after that
   base. This reproduces the collections in `schema.fcl` exactly, on the configurations tested.
2. **By parameter-set structure**, for flat configurations with no includes at all. Boards of the
   same type have identical key sets; the group is named after the longest common filename prefix.

`schema.fcl` is deliberately *not* used for grouping, though it is still compared as metadata. It is
absent from some exports entirely, and where present it is itself a versioned file that differs
between the two configurations being compared — so using one side's patterns to label the other's
files would be quietly wrong.

## Requirements

Python 3 and the `fhicl` extension module from `py-fhicl-py`. No third-party Python packages.

`fhicl.so` is located automatically, trying in order:

1. `import fhicl` — already on `sys.path` (e.g. after `spack load py-fhicl-py`).
2. `$PY_FHICL_PY_LIB`, if set — the directory containing `fhicl.so`.
3. `spack location -i py-fhicl-py`.
4. `/daq/software/spack_packages/py-fhicl-py/*/*/lib/fhicl.so`, newest first.

On the ICARUS DAQ machines this means the tool runs with no setup at all: `fhicl.so` is built for
Python 3.9, which is also the system Python, so step 4 finds it and no `spack load` is needed.

`fhicl.so` is a CPython extension and must match the running interpreter's ABI. If a future
`py-fhicl-py` is built against a different Python, run this tool with that interpreter — or fall
back to the `fhicl-dump` CLI in `/daq/software/spack_packages/fhicl-cpp/*/bin/`, which is
language-independent. The tool prints every location it tried when the import fails.

## Limitations

- **Directories only.** Fetching a configuration by name from the database is out of scope; use
  `conftool.py exportConfiguration` (or the existing `compare_configs.sh`) first.
- **`hashes.fcl` is skipped.** It is derived md5-per-file and flips on whitespace alone.
- **Non-`.fcl` files are ignored** (`boot.txt`, `environment.txt`, …).
- **Include resolution is value-based.** A key set to the same value it would have inherited is not
  counted as a local override. This is what makes the layering robust to `#include` appearing
  partway through a file, but it means a redundant restatement is invisible.
- **No filtering.** Known per-instance churn (metrics filenames, graphite namespaces,
  `process_name`) is reported like anything else. If it becomes noisy, an `--ignore` regex option is
  the natural addition.
