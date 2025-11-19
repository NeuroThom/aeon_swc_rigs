## Package layout

The Python package is a namespace package in this repository under `src/swc/aeon_rigs`:

- `base.py` – common base classes:
  - `BaseSchema`: Pydantic base model with Aeon JSON conventions
  - `Device`: base class for Aeon hardware devices
- `experiment.py` – minimal experiment metadata:
  - `Experiment`: workflow path, commit, repository URL
- `harp.py` – Harp device definitions:
  - `HarpDevice` and subclasses (different devices)
- `video.py` – camera devices:
  - `SpinnakerCamera`: FLIR/Spinnaker camera configuration
- `foraging.py` – foraging-specific devices:
  - `UndergroundFeeder`: Harp output expander with pellet delivery parameters
- `rig.py` – collated rig definition:
  - `Rig`: list of cameras, feeders, and a clock synchronizer
  - `main()`: generates a `rig.json` JSON schema file

---

## Key concepts

### `BaseSchema`

All schemas should derive from `BaseSchema`:

Additional fields can be added to new classes with any informative name and strict typing. A description should almost always be added to each new item:

```python
from swc.aeon_rigs.base import BaseSchema
from pydantic import Field

class MyClass(BaseSchema): # Create a new class derived from `BaseSchema` and inheriting it's fields.
    some_field: int = Field(description="An example field.") # Add a new field name "some_field", the type `int`, and a description.
```

### `Device`

`Device` is the base class for all hardware devices:

```python
from typing import ClassVar
from pydantic import Field
from swc.aeon_rigs.base import Device

class MyDevice(Device):
    device_type: ClassVar[str] = "MyDevice" # Class variable; see details below
    port_name: str = Field(description="Serial port or address.")
```

Important details:

- `device_type` is a class variable used as a discriminator in unions.
- When you define a field as a union of devices and mark it with
  `Field(discriminator="device_type")`, Pydantic will use `device_type` to dispatch to the correct subclass.

An example of a "Union" of devices can be found in the `Rig.clock_synchronizer` in the "Example `Rig` schema..." section below, which may be one of two different types of harp device.

---

### `Experiment` (base metadata)

`swc.aeon_rigs.experiment` defines a minimal metadata model:

```python
from pydantic import Field
from swc.aeon_rigs.base import BaseSchema

class Experiment(BaseSchema):
    workflow: str = Field(description="Path to the workflow running the experiment.")
    commit: str = Field(description="Commit hash of the experiment repo.")
    repository_url: str = Field(
        description="The URL of the git repository used to version experiment source code."
    )
```

Experiment repositories should subclass this to add their own configuration:

```python
from swc.aeon_rigs.experiment import Experiment as ExperimentBase
from my_experiment_pkg.rig import Rig
from my_experiment_pkg.task import TaskConfig

class Experiment(ExperimentBase):
    rig: Rig
    task: TaskConfig
```

This keeps versioning and provenance information consistent across experiments.

---

### Harp devices

`swc.aeon_rigs.harp` defines a hierarchy of Harp-based devices:

```python
from typing import ClassVar
from pydantic import Field
from swc.aeon_rigs.base import Device

class HarpDevice(Device):
    who_am_i: ClassVar[int] = Field(description="The unique identifier for the device type.")
    port_name: str = Field(examples=["COM"], description="The name of the device serial port.")
```

Concrete subclasses (defined devices) include (non-exhaustive):

- `HarpInputExpander`
- `HarpOutputExpander`
- `HarpClockSynchronizer`
- `HarpTimestampGeneratorGen3`
- `HarpAudioSwitch`
- `HarpSoundCard`
- `HarpRfidReader`

Each subclass:

- sets a specific `device_type` string (e.g. `"HarpSoundCard"`)
- sets the corresponding `who_am_i` code.

Experiment schemas can use these directly, or subclass them to add extra configuration fields.

---

### Cameras: `SpinnakerCamera`

`swc.aeon_rigs.video` provides a generic Spinnaker camera schema:

```python
from typing import ClassVar
from pydantic import Field
from swc.aeon_rigs.base import Device

class SpinnakerCamera(Device):
    device_type: ClassVar[str] = "SpinnakerCamera"
    serial_number: str = Field(
        examples=["00000"],
        description="The serial number of the camera.",
    )
    exposure_time: float = Field(
        default=1000,
        ge=100,
        description="The exposure time of the sensor, in microseconds.",
    )
    gain: float = Field(default=0, ge=0, description="The camera gain.")
    binning: int = Field(default=1, ge=1, description="The camera binning configuration.")
```

Experiment repos typically wrap or subclass this (e.g. to add tracking regions, triggers, etc.)
and then include those camera objects inside their `Rig` model.

---

### Foraging device: `UndergroundFeeder`

`swc.aeon_rigs.foraging` defines an underground feeder based on a Harp output expander:

```python
from pydantic import Field
from swc.aeon_rigs.harp import HarpOutputExpander

class UndergroundFeeder(HarpOutputExpander):
    pellet_delivery_retry_count: int = Field(
        default=2,
        ge=0,
        description="The number of times to retry a failed pellet delivery.",
    )
    pellet_delivery_timeout: float = Field(
        default=1,
        ge=0,
        description="The amount of time to wait for pellet detection before reporting a failure.",
    )
    wheel_radius: float = Field(
        default=-4.0,
        description="The radius of the wheel, in centimeters.",
    )
```

Experiment schemas can either use this directly or subclass it to specialize behaviour.

---


### Example `Rig` schema and JSON Schema generation

`swc.aeon_rigs.rig` includes a minimal example rig model that demonstrates how to
compose devices and export JSON Schema:

```python
import json
from pathlib import Path
from typing import Annotated, List, Union

from pydantic import Field
from swc.aeon_rigs.base import BaseSchema
from swc.aeon_rigs.foraging import UndergroundFeeder
from swc.aeon_rigs.video import SpinnakerCamera
from swc.aeon_rigs.harp import HarpClockSynchronizer, HarpTimestampGeneratorGen3


class Rig(BaseSchema):
    cameras: List[SpinnakerCamera]
    feeders: List[UndergroundFeeder]
    clock_synchronizer: Annotated[
        Union[HarpClockSynchronizer, HarpTimestampGeneratorGen3],
        Field(discriminator="device_type"),
    ]


def main():
    schema = Rig.model_json_schema(union_format="primitive_type_array")
    Path("rig.json").write_text(json.dumps(schema, indent=2))


if __name__ == "__main__":
    main()
```

Running this script with:

```bash
python -m swc.aeon_rigs.rig
```

produces a `rig.json` file with the JSON Schema for the `Rig` model. This is the same pattern used in experiment repos to generate schemas for more complex configurations.

---