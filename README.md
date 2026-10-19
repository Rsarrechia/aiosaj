# aiosaj

[![PyPI](https://img.shields.io/pypi/v/aiosaj)](https://pypi.org/project/aiosaj/)
[![Python](https://img.shields.io/pypi/pyversions/aiosaj)](https://pypi.org/project/aiosaj/)
[![tests](https://github.com/Rsarrechia/aiosaj/actions/workflows/tests.yml/badge.svg)](https://github.com/Rsarrechia/aiosaj/actions/workflows/tests.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

Async Python library to read live data from SAJ solar inverters, over either their
Ethernet (LAN) interface or their WiFi module. It was written for Home Assistant but
works in any asyncio application.

aiosaj continues the work of [fredericvl/pysaj](https://github.com/fredericvl/pysaj).

## Installation

```
pip install aiosaj
```

Requires Python 3.9 or newer.

## Usage

```python
import asyncio

from aiosaj import SAJ, Sensors


async def main():
    # WiFi module: wifi=True, with the module's web login (admin/admin by default).
    # Ethernet: SAJ("192.168.1.100") - no login needed.
    saj = SAJ("192.168.1.100", wifi=True, username="admin", password="admin")
    sensors = Sensors(wifi=True)

    if await saj.read(sensors):
        for sensor in sensors:
            if sensor.enabled:
                print(sensor.name, sensor.value, sensor.unit)
    else:
        print("Inverter not reachable (it switches off when there is no sunlight)")


asyncio.run(main())
```

- `read()` returns `True` when new values were read, and `False` when the inverter
  cannot be reached. SAJ inverters run on solar power, so this is normal at night.
- A wrong username or password raises `UnauthorizedException`. A response that is not
  inverter data raises `UnexpectedResponseException`.
- `sensors["total_yield"]` looks up a single sensor by name.
- `sensor.enabled` is `True` for every sensor your inverter actually reports. Sensors
  for channels your inverter does not have, such as a third PV string, stay disabled.
- `saj.serialnumber` holds the inverter's serial number after the first read, if the
  inverter's firmware provides it.

Use `Sensors(wifi=True)` together with `SAJ(..., wifi=True)`, and `Sensors()` with an
Ethernet connection.

## Sensors

### Both interfaces

| Sensor | Unit | Description |
| --- | --- | --- |
| `current_power` | W | Power being delivered right now |
| `today_yield` | kWh | Energy produced today |
| `total_yield` | kWh | Energy produced over the inverter's lifetime |
| `today_time` | h | Hours in operation today |
| `total_time` | h | Hours in operation over the inverter's lifetime |
| `total_co2_reduced` | kg | Lifetime CO₂ saving, as calculated by the inverter |
| `temperature` | °C | Inverter temperature |
| `state` | | Run state: `Normal`, `Waiting`, `Not connected`, `Error` or `Upgrading` |

### Ethernet only

| Sensor | Unit | Description |
| --- | --- | --- |
| `today_max_current` | W | Highest power output reached today |

### WiFi only

| Sensor | Unit | Description |
| --- | --- | --- |
| `pv1_voltage`, `pv2_voltage`, `pv3_voltage` | V | Voltage of each PV input |
| `pv1_current`, `pv2_current`, `pv3_current` | A | Current of each PV input |
| `pv1_string1_current` … `pv3_string4_current` | A | Current of each string on each PV input |
| `grid_frequency` | Hz | Grid frequency |
| `line1_voltage`, `line2_voltage`, `line3_voltage` | V | Grid voltage per phase |
| `line1_current`, `line2_current`, `line3_current` | A | Grid current per phase |
| `bus_voltage` | V | Internal DC bus voltage |

Older WiFi module firmware only reports the sensors that both interfaces have.

## How values behave

- Values are only updated while the inverter's state is `Normal`. While it is starting
  up, waiting or in error, live values such as power and voltage become `None`, and
  the energy and time counters keep their last value.
- `total_yield`, `total_time` and `total_co2_reduced` never go down, and `today_yield`
  and `today_time` only go back to zero when a new day starts. This keeps energy
  statistics, such as Home Assistant's Energy dashboard, free of false spikes.
- A reading that is out of range or clearly wrong is skipped, and the sensor keeps
  its previous value.

## Migrating from pysaj

The API is the same as pysaj. Install `aiosaj` and change your imports:

```python
# before
from pysaj import SAJ, Sensors
# after
from aiosaj import SAJ, Sensors
```

See [CHANGELOG.md](CHANGELOG.md) for everything that changed since pysaj 0.0.16.

## Credits

aiosaj builds on [pysaj](https://github.com/fredericvl/pysaj) by @fredericvl, which was
in turn based on the pysma library by @kellerza.

## License

MIT, see [LICENSE](LICENSE).
