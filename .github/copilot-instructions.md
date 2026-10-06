# Repository instructions

## Build, test, and run

- The CI workflow targets Python 3.13.12, installs dependencies from `requirements.txt`, and runs pytest. Install dependencies with `python -m pip install -r requirements.txt`.
- Run the complete test suite with `python -m pytest -q`.
- Run a focused test file with `python -m pytest -q tests/test_onoff.py`, `python -m pytest -q tests/test_predictive_onoff.py`, or `python -m pytest -q tests/test_my_first_script.py`.
- Run one test by node ID, for example: `python -m pytest -q tests/test_onoff.py::test_safety_cutoff`.
- GitHub Actions separately runs those three test files on pushes to `assignment` and `solution`; use the matching test file when validating a change to one of those areas.
- Run a scenario from the repository root with `python main.py --scenario scenarios/door_open.yaml` or `python main.py --scenario scenarios/cold_morning.yaml`. The runner writes timestamped CSV logs to `outputs/logs/` and figures to `outputs/figures/`.

## Architecture

- `main.py` is the command-line entry point. It accepts a YAML scenario path and delegates execution to `scenarios/runner.py`.
- `utils/config.py` loads scenario YAML into nested dataclasses (`EnvConfig`, `SensorConfig`, `ControllerConfig`, `ModelConfig`, `SimConfig`). Scenario files use those same top-level sections: `env`, `sensor`, `controller`, `model`, and `sim`.
- `scenarios/runner.py` assembles and advances the simulation: it uses `Environment` for outdoor temperature, `TempSensor` and the sensor filters for measurements, chooses a controller from `controller.type`, advances the room with `step_room`, and records data for CSV/plot output.
- `simulations/` contains the outside-temperature and room thermal models; `sensors/` contains noisy/dropout temperature measurements and filtering; `controllers/` contains the simple and predictive thermostat strategies; `plotting/` turns the runner's log dictionary into saved figures.
- Random sensor and process behavior is routed through `utils/rng.py`; the runner seeds it from `sim.seed` so a scenario is reproducible.

## Codebase conventions

- Controller APIs intentionally differ: `OnOffThermostat.update` takes only the measured temperature, while `PredictiveOnOff.update` takes the filtered temperature and `dt`. The runner branches on `controller.type` and handles each API separately; keep that dispatch and its diagnostics consistent when changing controllers.
- Both controllers use integer heater states (`0` off, `1` on). The deadband limits are centered on the setpoint (`setpoint ± deadband / 2`), the controller retains its current state inside the band, and `safety_high` is an overriding shutoff threshold.
- Keep scenario configuration aligned across the YAML keys, the matching dataclass fields in `utils/config.py`, and their use in `scenarios/runner.py`. Existing scenarios are the runnable examples.
- The simulation log is a dictionary of parallel lists keyed by names such as `t`, `T_true`, `T_meas`, `heater`, and `T_pred`; plotting functions consume these keys, including predictive diagnostics. Update logging and plot consumers together when changing that schema.
- The assignment tests specify the controller contracts and the `my_first_script.py` behavior. The GitHub Actions workflow invokes these test files independently, so preserve their import paths and public call signatures.
