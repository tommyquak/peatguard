# PeatGuard

Satellite radar for village-led peatland restoration in Central Kalimantan, Indonesia.

Drained tropical peat dries out, sinks and burns. Blocking the old drainage canals rewets it, but
restoration crews need to know which canals to block first and funders need evidence that the
blocks worked. PeatGuard turns radar observations of the peat surface into a ranked list of
canal-blocking sites, dispatches them to village crews through a field app, and pays the crews
for verified work. The study area is the Sebangau landscape and the former Mega Rice Project.

This repository is the public showcase of the processing core: the NISAR L-band ingest,
time-series and validation code, with its test suite. The operator web console and the field
app are being rebuilt and are not included yet. The live site is [peatguard.com](https://peatguard.com).

## What is in here

```
pipeline/
  src/peatguard/
    ingest/nisar_gslc.py               Stage only the AOI window of a ~20 GiB NISAR GSLC via HTTP range reads
    insar/nisar_gslc.py                5 m wrapped interferograms and local coherence from GSLC pairs
    insar/nisar_gunw.py                QA layers from mission GUNW products (phase, coherence, ionosphere)
    insar/nisar_fine_product.py        Compact quality-only coherence product, block-wise to UInt8 COG
    insar/nisar_validation.py          Checks fine GSLC interferograms against mission GUNW
    timeseries/nisar_gunw.py           Explicit correction variants and a reproducible reference patch
    timeseries/nisar_commissioning.py  QA for short stacks: correction sensitivity, ionosphere uncertainty
    timeseries/local_contrast.py       Target-minus-background contrasts on fixed spatial support
    analysis/                          Baseline audit, grouped contrast validation, risk sensitivity
    export/cog.py                      Cloud-Optimized GeoTIFF export
  scripts/                             Command-line drivers for the site contrast and baseline tables
  tests/                               160 tests on synthetic fixtures; no credentials or downloads needed
```

## Quick start

```bash
cd pipeline
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
pytest
```

The tests build small synthetic GSLC, GUNW and raster fixtures, so they run offline in a few
seconds. Working with real NISAR data needs a free NASA Earthdata login in `~/.netrc`; the ingest
module then streams only the byte ranges covering your area of interest instead of whole
multi-gigabyte granules.

## What the radar can and cannot tell us here

Most of the work behind this code was finding out which measurements hold up over wet, forested
tropical peat. The results below are our own processing and have not been peer reviewed. Tags:
[E] established in the cited literature, [S] standard inference, [H] our hypothesis.

1. **Sentinel-1 C-band cannot measure peat subsidence over this area.** We ran a full ISCE2
   topsStack and MintPy SBAS time series (45 dates, 249 interferograms) over Sebangau and compared
   the velocity with the independent ALOS-2 L-band subsidence map of Tay et al. (2025). Across
   about 169,000 pixels the correlation was about -0.03. A faithful rerun of the C-band study
   block used by Umarhadi and Siegert (2026) was more coherent but still uncorrelated (about -0.04),
   because the expected signal (about 9.6 mm/yr spread) sits below the C-band noise (about
   22.5 mm/yr). That paper's accuracy figure comes from internal cross-validation, which is not an
   independent check. [S]
2. **L-band is required, and even ALOS-2 ScanSAR is marginal.** A 93-scene ALOS-2 stack processed
   on an HPC cluster into 1,650 interferograms sat at the coherence floor over the peat. It gives
   a bounded regional mean, not a per-pixel rate, and none of its usable pixels overlap the area
   Tay et al. validated. [S]
3. **NISAR sees the peat, but subsidence is not yet usable.** NISAR's 12-day L-band pairs keep
   coherence over the peat where the other sensors lose it. Our pre-registered reference
   ensemble still failed its gates: there is no stable mineral ground within 60 km to tie the
   measurement to, and per-pair residuals of 20 to 40 mm flip sign between pairs. So this code
   reports gated contrasts and quality flags, and it never publishes a subsidence rate that has not
   passed them. [S]
4. **Subsidence matters because it is carbon.** Peat surface lowering under drainage tracks
   carbon loss closely (Hooijer et al., 2012), which is why a trustworthy rate is worth this much
   effort. [E]

A related exploratory result, a per-pixel dry-season moisture signal from NISAR backscatter, is
in the team's NASA Space Apps 2026 project,
[Dry Season Waltz](https://github.com/tommyquak/GarugonsSpaceApps).

## Design rules the code follows

- **Validation data stays independent.** Tay et al. (2025) is a benchmark only. No bias, scale or
  datum fit to it is allowed anywhere in the code, so agreement with it means something.
- **A short stack is not a rate.** NISAR mission corrections (ionosphere, troposphere, solid Earth
  tides) are applied as explicit, comparable variants; the code measures how much each one and the
  choice of reference patch moves the answer, and it never annualises a 24-day displacement.
- **Contrasts, not absolute motion.** Without stable ground nearby, the code compares each site with
  its own surroundings on fixed support, so a shared phase datum cancels. Validation groups sites
  spatially to avoid pseudoreplication (Roberts et al., 2017).
- **Provenance travels with the numbers.** Outputs record SHA-256 hashes of the inputs and the
  windows actually read, so any table can be traced back to the data that produced it.

## References

- Tay, C., Jovani-Sancho, A. J., Yulianti, L., Evans, C. et al. (2025). Satellite radar advances
  carbon emissions accountability over tropical peat. *Communications Earth & Environment* 6.
  doi:10.1038/s43247-025-02926-6
- Umarhadi, D. A. and Siegert, F. (2026). Monitoring degraded tropical peatland subsidence by
  integrating SBAS InSAR and machine learning. *Environmental Research Communications* 8, 025025.
  doi:10.1088/2515-7620/ae43ab
- Hooijer, A., Page, S., Jauhiainen, J., Lee, W. A. et al. (2012). Subsidence and carbon loss in
  drained tropical peatlands. *Biogeosciences* 9, 1053-1071. doi:10.5194/bg-9-1053-2012
- Roberts, D. R. et al. (2017). Cross-validation strategies for data with temporal, spatial,
  hierarchical, or phylogenetic structure. *Ecography* 40, 913-929. doi:10.1111/ecog.02881

## Recognition

- 2nd place, PDMA Global Student Innovation Challenge 2026 (global finals, Atlanta)
- First runner-up, Hult Prize Singapore National Finals 2026
- 3rd place, Adaptation and Resilience track, Global Sustainability Challenge 2025-26
- Stanford EcoGlobal Program 2026

## How this repository is maintained

This repository is generated from PeatGuard's private development repo by an allowlist export
that refuses to publish anything matching a credential or private-host pattern. Changes arrive
here as pull requests from that export. Issues are welcome.

## License

MIT. See [LICENSE](LICENSE). NISAR and ALOS-2 data are distributed by NASA ASF under their own
terms; no satellite imagery or derived data products are included in this repository.
