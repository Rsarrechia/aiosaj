# Changelog

All notable changes to aiosaj (formerly the maintained pysaj fork) are
documented here. This project follows
[Semantic Versioning](https://semver.org/).

## [0.2.1] - 2026-10-19

### Changed
- README rewritten as a user guide: installation, usage, the sensors each
  interface exposes and how their values behave. No code changes.

## [0.2.0] - 2026-10-19

### Changed
- **Renamed to aiosaj.** The package is published on PyPI as `aiosaj` and
  imported as `aiosaj`; replace `import pysaj` with `import aiosaj`. The new
  import name lets it be installed alongside the original `pysaj`, which Home
  Assistant core still requires.
- Only a record whose run state is `Normal` is trusted. An empty, unparseable
  or unrecognised state now gates the record like a known non-normal state:
  live readings become unknown and counters keep their last value. A record
  with no state field at all is still applied.
- **Upgrading from upstream 0.0.16:** Ethernet/LAN values are now floats
  (`1000.0`) instead of the XML text (`"1000"`), matching the WiFi interface.

### Fixed
- An implausible jump in a lifetime counter is only accepted once the
  confirming readings agree with each other. A run of ever higher corrupt
  values (50000 -> 70000 -> 90000) no longer counts as confirmation.
- `today_yield` and `today_time` can no longer go down within the same day; a
  drop is only accepted once the date has changed.

### Added
- Linting with ruff in CI.

## [0.1.3] - 2026-08-18

### Fixed
- **The Ethernet/LAN interface was broken since 0.1.0.** `real_time_data.xml`
  spells the run state out (`<state>Normal</state>`), so mapping it through
  `MAPPER_STATES` produced `Unknown(Normal)`, the run-state gate never let a
  reading through, and every LAN sensor stayed unknown while `read()` still
  reported success. Numeric codes are still mapped; any other text is taken as
  the state itself, and only the four known non-normal states gate readings, so
  an unrecognised value can no longer blank a sensor.
- The XML is already in real units (`<temp>7.0</temp>`,
  `<e-total>2293.08</e-total>`), so the CSV scaling factors are no longer
  applied to it. This reverts the change the 0.1.2 notes described as a fix: it
  was a regression, and 0.0.16 was right to apply factors only to the CSV.
- A channel the inverter does not have is a literal `-` on the LAN interface,
  not `65535`, and now stages as absent so it gets no entity.
- The device info page is read only until the serial number is known, and a
  failure to read it no longer stops the inverter being read. Newer firmware has
  dropped that page entirely, which broke setup outright; it carries nothing but
  the serial number. A 401 still surfaces as an authentication error.

## [0.1.2] - 2026-08-18

### Fixed
- A cumulative counter's growth allowance is never smaller than one step of its
  own resolution. `total_time` is `/10` scaled with a 1.2h/h limit, so at any
  poll interval below five minutes every normal 0.1h increment was held back as
  an implausible jump and logged as one.
- Warnings raised per record are throttled: the inverter run state, out of range
  fields, accepted jumps, malformed records and daily-above-lifetime rejections.
  Throttle state is keyed per message, so one recurring rejection no longer
  suppresses a different one or inflates its suppressed count.
- A sensor whose field is present but whose value is unusable (`grid_frequency`
  reads 0Hz until the module synchronises) is exposed again, so the consumer
  still creates its entity when the first read lands in such a window. Channels
  the inverter reports as not available (`65535`) stay hidden.

### Documentation
- The LAN XML tag names for the added sensors are corrected: `v-pv1`, `i-pv11`
  through `i-pv34`, `v-bus`, `Vac_l1`-`Vac_l3`, `Iac_l1`-`Iac_l3` and `Freq1`,
  from two independently posted `real_time_data.xml` dumps. They are documented
  but deliberately not wired up until confirmed against a device.

### Note
- Since 0.1.0 the Ethernet/XML path applies each sensor's `factor`, where it
  previously stored the raw text. Values from `real_time_data.xml` are now
  scaled the same way as the WiFi ones (e.g. temperature `250` reads as 25.0).

## [0.1.1] - 2026-08-17

### Changed
- Modern packaging: `pyproject.toml` (PEP 621) replaces `setup.py`; test
  dependencies moved to a `test` extra; added a GitHub Actions test workflow.
- `pysaj.__version__` is now derived from the installed package metadata.
- Tests no longer depend on `aioresponses` (which does not support current
  `aiohttp`); the HTTP layer is mocked directly, so the suite runs green
  against the `aiohttp` Home Assistant ships.

## [0.1.0]

### Fixed
- Lifetime counters (`total_yield`, `total_time`, `total_co2_reduced`) can no
  longer decrease, and implausible jumps are held until confirmed. This fixes a
  Home Assistant meter-reset spike where a single bad reading was booked as a
  full counter's worth of production in one hour.
- Malformed status records (wrong field count, wait markers, multi-line,
  partial) are discarded instead of being indexed into and misparsed.
- Negative temperatures are read correctly (previously discarded due to a
  `name`/`key` mix-up).

### Added
- `65535` is honoured as the module's not-available sentinel; per-sensor range
  checks; readings are ignored while the inverter run state is not `Normal`.
- 26 previously dropped WiFi channels: PV string voltages/currents, grid
  frequency, per-phase AC voltage/current, and DC bus voltage.
- Throttling of repeated counter-rejection warnings (at most once per five
  minutes) so a persistent cause cannot flood the log.
- Test suite (`tests/test_pysaj.py`).

This fork is based on [fredericvl/pysaj](https://github.com/fredericvl/pysaj)
(last released as 0.0.16), itself derived from work by kellerza. Licensed MIT.
