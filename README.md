# ❗ OpenTherm in ESPHome core

Starting with ESPHome 2024.11 this component is included in core! 🎉 There is no need to reference this repository for
normal usage scenarios. Use [official documentation](https://esphome.io/components/opentherm.html) to configure your
OpenTherm bridge.

If you want to test bleeding edge features and fixes, read the Usage section below.

# OpenTherm Component for ESPHome

OpenTherm (OT) is a standard communications protocol used in central heating systems for the communication between
central heating appliances and a thermostatic controller. As a standard, OpenTherm is independent of any single
manufacturer. A controller from manufacturer A can in principle be used to control a boiler from manufacturer B.

Since OpenTherm doesn't operate in a standard voltage range, special hardware is required. You can choose from several
ready-made adapters or roll your own:

- [DIYLESS Master OpenTherm Shield](https://diyless.com/product/master-opentherm-shield)
- [Ihor Melnyk's OpenTherm Adapter](http://ihormelnyk.com/opentherm_adapter)
- [Jiří Praus' OpenTherm Gateway Arduino Shield](https://www.tindie.com/products/jiripraus/opentherm-gateway-arduino-shield/)

‼️ As of now, this component acts only as an OpenTherm master (for example, a thermostat or controller) and not as a
slave or gateway. Your existing thermostat is not usable while you use ESPHome with this component to control your
boiler.

There are plans to add support for a gateway mode, but I don't have any timeline to share at the moment.

## Quick glossary

- CH: Central Heating
- DHW: Domestic Hot Water

## Usage

As I've mentioned before, this component is now part of ESPHome core. Most users don't need this repository. Just follow
the [official documentation](https://esphome.io/components/opentherm.html) to setup your bridge.

### Testing unstable versions

If you want to test unstable versions of the code, choose your branch first:

* `main` — contains the code that was merged to ESPHome core. No need to reference this branch in your config really.
* `develop` — bleeding edge changes that are not yet proposed to be merged. Individual commits may not even compile.
  **Use commit hashes or known tags to reference individual commits from this branch!**
* `pr_*` — Changes that correspond to open PRs to ESPHome core.

After you've chosen your branch, or individual commit, add this repository as external component to your config:

```yaml
external_components:
  source: github://olegtarasov/esphome-opentherm[@<branch or tag>]
  refresh: 0s
```

## Declaring the OpenTherm hub

You need to declare the OpenTherm hub in your configuration. Note that most OpenTherm adapters label `in` and
`out` pins relative to themselves; this component labels its `in` and `out` pins relative to the microcontroller
ESPHome runs on. As such, your bridge's `in` pin becomes the hub's `out` pin and vice versa.

```yaml
opentherm:
  in_pin: GPIOXX
  out_pin: GPIOXX
```

### Configuration variables

- `in_pin` (**Required**, number): The pin of the OpenTherm hardware bridge which is usually labeled ``out`` on the
  board.
- `out_pin` (**Required**, number): The pin of the OpenTherm hardware bridge which is usually labeled ``in`` on the
  board.
- `sync_mode` (**Optional**, boolean, default **false**): Synchronous communication mode prevents other components
  from disabling interrupts while we are talking to the boiler. Enable if you experience a lot of random intermittent
  invalid response errors (very likely to happen while using Dallas temperature sensors).
- `id` (**Optional**): Manually specify the ID used for code generation.  Required if you have multiple buses.

#### Optional boiler-specific configuration

Some boilers require certain OpenTherm messages to be sent by thermostat on initialization in order to work correctly.
You can use the following settings in hub configuration to make your particular boiler happy.

<!-- BEGIN schema_docs:setting -->
- `controller_product_type` (**Optional**, byte [0-255], OpenTherm message id `126` high byte): Controller product type
- `controller_product_version` (**Optional**, byte [0-255], OpenTherm message id `126` low byte): Controller product
  version
- `opentherm_version_controller` (**Optional**, float, OpenTherm message id `124`): Version of OpenTherm implemented by
controller
- `controller_configuration` (**Optional**, byte [0-255], OpenTherm message id `2` high byte): Controller configuration
- `controller_id` (**Optional**, byte [0-255], OpenTherm message id `2` low byte): Controller ID code
<!-- END schema_docs:setting -->

#### Automations

- `before_send` (**Optional**) An automation to perform on OpenTherm message before it is sent to the boiler.
- `before_process_response` (**Optional**) An automation to perform on boiler response before it is processed.

See [below](#on-the-fly-message-editing) for details.

### Note abut sync mode

The use of some components (like Dallas temperature sensors) may result in lost frames and protocol warnings from
OpenTherm. Since OpenTherm is resilient by design and transmits its messages in a constant loop, these dropped frames
don't usually cause any problems. Still, if you want to decrease the number of protocol warnings in your logs, you can
enable `sync_mode` which will block ESPHome's main application loop until a single conversation with the boiler is
complete. This can greatly reduce the number of dropped frames, but usually won't eliminate them entirely. With
`sync_mode` enabled, in some cases, ESPHome's main application loop may be blocked for longer than is recommended,
resulting in warnings in the logs. If this bothers you, you can adjust ESPHome's log level by adding the following to
your configuration:

```yaml
logger:
  logs:
    component: ERROR
```

## Usage as a thermostat

The most important function for a thermostat is to set the boiler temperature setpoint. This component has three ways
to provide this input: using a Home Assistant sensor from which the setpoint can be read, using a
[number](https://esphome.io/components/number/), or defining an output to which other components can write.
For most users, the last option is the most useful one, as it can be combined with the
[PID](https://esphome.io/components/climate/pid.html) component to create a thermostat that works as you would expect
a thermostat to work. See thermostat example further in this readme.

### Numerical input

There are three ways to set an input value:

- As an input sensor, defined in the hub configuration:

  ```yaml
  opentherm:
    t_set: setpoint_sensor

  sensor:
    - platform: homeassistant
      id: setpoint_sensor
      entity_id: sensor.boiler_setpoint
  ```

  This can be useful if you have an external thermostat-like device that provides the setpoint as a sensor.

- As a number:

  ```yaml
  number:
    - platform: opentherm
      t_set:
        name: Boiler Setpoint
  ```

  This is useful if you want full control over your boiler and want to manually set all values.

- As an output:

  ```yaml
  output:
    - platform: opentherm
      t_set:
        id: setpoint
  ```

  This is especially useful in combination with the PID Climate component:

  ```yaml
  climate:
    - platform: pid
      heat_output: setpoint
      # ...
  ```

For the output and number variants, there are four more properties you can configure beyond those included in the
output and number components by default:

- `min_value` (float): The minimum value. For a number this is the minimum value you are allowed to input.
For an output this is the number that will be sent to the boiler when the output is at 0%.
- `max_value` (float): The maximum value. For a number this is the maximum value you are allowed to input.
For an output this is the number that will be sent to the boiler when the output is at 100%.
- `auto_max_value` (boolean): Automatically configure the maximum value to a value reported by the boiler.
Not available for all inputs.
- `auto_min_value` (boolean): Automatically configure the minimum value to a value reported by the boiler.
Not available for all inputs.

The following inputs are available:

<!-- BEGIN schema_docs:input -->
- `t_set`: Control setpoint: temperature setpoint for the boiler's supply water (°C)
  Default `min_value`: 0
  Default `max_value`: 100
  Supports `auto_max_value`
- `t_set_ch2`: Control setpoint 2: temperature setpoint for the boiler's supply water on the second heating circuit (°C)
  Default `min_value`: 0
  Default `max_value`: 100
  Supports `auto_max_value`
- `cooling_control`: Cooling control signal (%)
  Default `min_value`: 0
  Default `max_value`: 100
- `t_dhw_set`: Domestic hot water temperature setpoint (°C)
  Default `min_value`: 0
  Default `max_value`: 127
  Supports `auto_min_value`
  Supports `auto_max_value`
- `max_t_set`: Maximum allowable CH water setpoint (°C)
  Default `min_value`: 0
  Default `max_value`: 127
  Supports `auto_min_value`
  Supports `auto_max_value`
- `t_room_set`: Current room temperature setpoint (informational) (°C)
  Default `min_value`: -40
  Default `max_value`: 127
- `t_room_set_ch2`: Current room temperature setpoint on CH2 (informational) (°C)
  Default `min_value`: -40
  Default `max_value`: 127
- `t_room`: Current sensed room temperature (informational) (°C)
  Default `min_value`: -40
  Default `max_value`: 127
- `max_rel_mod_level`: Maximum relative modulation level (%)
  Default `min_value`: 0
  Default `max_value`: 100
- `otc_hc_ratio`: OTC heat curve ratio (°C)
  Default `min_value`: 0
  Default `max_value`: 127
  Supports `auto_min_value`
  Supports `auto_max_value`
<!-- END schema_docs:input -->

### Switch

Switches are available to allow manual toggling of any of the following seven status codes:

- `ch_enable`: Central Heating enabled
- `dhw_enable`: Domestic Hot Water enabled
- `cooling_enable`: Cooling enabled
- `otc_active`: Outside temperature compensation active
- `ch2_active`: Central Heating 2 active
- `summer_mode_active`: Summer mode active
- `dhw_block`: Block DHW

If you do not wish to have switches, the same values can be permanently set in the hub configuration, like so:

```yaml
opentherm:
  ch_enable: true
  dhw_enable: true
```

This is useful when you'd never want to toggle it after the initial configuration.

The default values for these configuration variables are listed below.

To enable central heating and cooling, the flag is only sent to the boiler if the following conditions are met:

- the flag is set to true in the hub configuration,
- the switch is on (if configured),
- the setpoint or cooling control value is not 0 (if configured)

For domestic hot water and outside temperature compensation, only the first two conditions are necessary.

The last point ensures that central heating is not enabled if no heating is requested as indicated by a setpoint of 0.
If you use a number as the setpoint input and use a minimum value higher than 0, you **must** use the ``ch_enable``
switch to turn off your central heating. In such a case, the flag will be set to true in the hub configuration and the
setpoint is always larger than 0, so including a switch is the only way you can turn off central heating. (This also
holds for cooling and CH2.)

### Binary sensor

The component can report boiler status on several binary sensors. The *Status* sensors are updated in each message
cycle, while the others are only set during initialization, as they are unlikely to change without restarting the
boiler.

<!-- BEGIN schema_docs:binary_sensor -->
- `fault_indication`: Status: Fault indication
- `ch_active`: Status: Central Heating active
- `dhw_active`: Status: Domestic Hot Water active
- `flame_on`: Status: Flame on
- `cooling_active`: Status: Cooling active
- `ch2_active`: Status: Central Heating 2 active
- `diagnostic_indication`: Status: Diagnostic event
- `electricity_production`: Status: Electricity production
- `dhw_present`: Configuration: DHW present
- `control_type_on_off`: Configuration: Control type is on/off
- `cooling_supported`: Configuration: Cooling supported
- `dhw_storage_tank`: Configuration: DHW storage tank
- `controller_pump_control_allowed`: Configuration: Controller pump control allowed
- `ch2_present`: Configuration: CH2 present
- `water_filling`: Configuration: Remote water filling
- `heat_mode`: Configuration: Heating or cooling
- `dhw_setpoint_transfer_enabled`: Remote boiler parameters: DHW setpoint transfer enabled
- `max_ch_setpoint_transfer_enabled`: Remote boiler parameters: CH maximum setpoint transfer enabled
- `dhw_setpoint_rw`: Remote boiler parameters: DHW setpoint read/write
- `max_ch_setpoint_rw`: Remote boiler parameters: CH maximum setpoint read/write
- `service_request`: Service required
- `lockout_reset`: Lockout Reset
- `low_water_pressure`: Low water pressure fault
- `flame_fault`: Flame fault
- `air_pressure_fault`: Air pressure fault
- `water_over_temp`: Water overtemperature
<!-- END schema_docs:binary_sensor -->

### Sensor

The boiler can also report several numerical values, which are available through sensors. Your boiler may not support
all of these values, in which case there won't be any value published to that sensor. The following sensors are
available:

<!-- BEGIN schema_docs:sensor -->
- `rel_mod_level`: Relative modulation level (%)
- `ch_pressure`: Water pressure in CH circuit (bar)
- `dhw_flow_rate`: Water flow rate in DHW circuit (l/min)
- `t_boiler`: Boiler water temperature (°C)
- `t_dhw`: DHW temperature (°C)
- `t_outside`: Outside temperature (°C)
- `t_ret`: Return water temperature (°C)
- `t_storage`: Solar storage temperature (°C)
- `t_collector`: Solar collector temperature (°C)
- `t_flow_ch2`: Flow water temperature CH2 circuit (°C)
- `t_dhw2`: Domestic hot water temperature 2 (°C)
- `t_exhaust`: Boiler exhaust temperature (°C)
- `fan_speed`: Boiler fan speed (RPM)
- `fan_speed_setpoint`: Boiler fan speed setpoint (RPM)
- `flame_current`: Boiler flame current (µA)
- `burner_starts`: Number of starts burner
- `ch_pump_starts`: Number of starts CH pump
- `dhw_pump_valve_starts`: Number of starts DHW pump/valve
- `dhw_burner_starts`: Number of starts burner during DHW mode
- `burner_operation_hours`: Number of hours that burner is in operation
- `ch_pump_operation_hours`: Number of hours that CH pump has been running
- `dhw_pump_valve_operation_hours`: Number of hours that DHW pump has been running or DHW valve has been opened
- `dhw_burner_operation_hours`: Number of hours that burner is in operation during DHW mode
- `t_dhw_set_ub`: Upper bound for adjustment of DHW setpoint (°C)
- `t_dhw_set_lb`: Lower bound for adjustment of DHW setpoint (°C)
- `max_t_set_ub`: Upper bound for adjustment of max CH setpoint (°C)
- `max_t_set_lb`: Lower bound for adjustment of max CH setpoint (°C)
- `t_dhw_set`: Domestic hot water temperature setpoint (°C)
- `max_t_set`: Maximum allowable CH water setpoint (°C)
- `oem_fault_code`: OEM fault code ()
- `oem_diagnostic_code`: OEM diagnostic code ()
- `max_capacity`: Maximum boiler capacity (KW) (kW)
- `min_mod_level`: Minimum modulation level (%)
- `opentherm_version_device`: Version of OpenTherm implemented by device ()
- `device_type`: Device product type ()
- `device_version`: Device product version ()
- `device_id`: Device ID code ()
- `otc_hc_ratio_ub`: OTC heat curve ratio upper bound ()
- `otc_hc_ratio_lb`: OTC heat curve ratio lower bound ()
<!-- END schema_docs:sensor -->

#### Fan speed sensor

An issue was raised about `fan_speed` sensor giving wonky values: https://github.com/olegtarasov/esphome-opentherm/issues/12.
It turned out that originally this library was using unsigned 16-bit integer to interpret fan speed, and it didn't work
for some boilers. OpenTherm specification suggests that 8-bit integer should be used, with another 8 bits being a
separate sensor, `fan_speed_setpoint`. Tests have also shown that for those boilers value interpreted as 8-bit integer
should be further multiplied by 60 to obtain final RPM value.

I decided to modify `fan_speed` sensor to work as 8-bit integer, performing the multiplication automatically, because
it's closer to OpenTherm specification.

Obviously, this breaks `fan_speed` sensor for boilers that encode values as 16-bit integers. In order to fix this, we
added `data_type` property which allows to override sensor data type.

So if you configured a `fan_speed` sensor, but suspect that it gives the wrong values, try to reconfigure it as follows:

```yaml
sensor:
  - platform: opentherm
    fan_speed:
      name: "Boiler fan speed"
      data_type: "u16" # overrides the default u8_lb_60 message
```

### On-the-fly message editing

Some boilers use non-standard message ids and formats. For example, [it's known](https://github.com/olegtarasov/esphome-opentherm/issues/11)
that Daikin D2C boiler uses message id `162` instead of `56` to set target DHW temperature. In order to accomodate all
sorts of non-standard behavior, I introduced two automations that allow editing the low-level OpenTherm message:

- `before_send`: fired just before the fully formed message is sent to the boiler. When you use a lambda, the message
  is passed by reference as `x`.
- `before_process_response`: fired when response message is received from the boiler and is about to be processed.
  When you use a lambda, the message is passed by reference as `x`.

This allows to make arbitrary alterations to any message. Here is an example of overriding message id for DHW setpoint
for Daikin D2C boiler:

```yaml
opentherm:
  # Usual hub config
  before_send:
      then:
        - lambda: |-
            if (x.id == 56) { // 56 is standard message id for DHW setpoint
              x.id = 162;     // message is passed by refence, so we can change anything, including message id
            }
  before_process_response:
    then:
      - lambda: |-
          if (x.id == 162) { // We substitute the original id back, so that esphome is not confused.
            x.id = 56;
          }
```

You can check the [OpenthermData reference](https://esphome.io/api/structesphome_1_1opentherm_1_1_opentherm_data) for
the list of all available fields.

# Examples

## Minimal example with numeric input

```yaml
# An extremely minimal configuration which only enables you to set the boiler's
# water temperature setpoint as a number.

opentherm:
  in_pin: GPIOXX
  out_pin: GPIOXX
  ch_enable: true

number:
  - platform: opentherm
    t_set:
      name: "Boiler Control setpoint"
```

## Basic PID thermostat
```yaml
# A basic thremostat for a boiler with a single central heating circuit and
# domestic hot water. It reports the flame, CH and DHW status, similar to what
# you would expect to see on a thermostat and also reports the internal boiler
# temperatures and the current modulation level. The temperature is regulated
# through a PID Climate controller and the current room temperature is retrieved
# from a sensor in Home Asisstant.

# This configuration should meet most needs and is the recommended starting
# point if you just want a thermostat with an external temperature sensor.

opentherm:
  in_pin: GPIOXX
  out_pin: GPIOXX
  dhw_enable: true    # Note that when we specify an input in hub config with a static value, it can't be
                      # changed without uploading new firmware. If you want to be able to turn things on or off,
                      # use a switch (see the ch_enable switch below).
                      # Also note that when we define an input as a switch (or use other platform), we don't need
                      # to set it at hub level.

output:
  - platform: opentherm
    t_set:
      id: t_set
      min_value: 20
      max_value: 65
      zero_means_zero: true

sensor:
  - platform: opentherm
    rel_mod_level:
      name: "Boiler Relative modulation level"
    t_boiler:
      name: "Boiler water temperature"
    t_ret:
      name: "Boiler Return water temperature"

  - platform: homeassistant
    id: ch_room_temperature
    entity_id: sensor.temperature
    filters:
      # Push room temperature every second to update PID parameters
      - heartbeat: 1s

binary_sensor:
  - platform: opentherm
    ch_active:
      name: "Boiler Central Heating active"
    dhw_active:
      name: "Boiler Domestic Hot Water active"
    flame_on:
      name: "Boiler Flame on"
    fault_indication:
      name: "Boiler Fault indication"
      entity_category: diagnostic
    diagnostic_indication:
      name: "Boiler Diagnostic event"
      entity_category: diagnostic

switch:
  - platform: opentherm
    ch_enable:
      name: "Boiler Central Heating enabled"
      restore_mode: RESTORE_DEFAULT_ON

climate:
  - platform: pid
    name: "Central heating"
    heat_output: t_set
    default_target_temperature: 20
    sensor: ch_room_temperature
    control_parameters:
      kp: 0.4
      ki: 0.004
```

# References

This component was forked from Arthur Rump's ``esphome-opentherm`` component, which now seems to be abandoned. I
replaced the underlying OpenTherm library with code form Jiří Praus. I also did a lot of refactoring to bring the code
closer to ESPHome coding standard.

- [Original Arthur Rump's repository](https://github.com/arthurrump/esphome-opentherm)
- [arduino-opentherm project by Jiří Praus](https://github.com/jpraus/arduino-opentherm)

There is also my blog post with more background details and reasoning for automating an OpenTherm boiler with ESPHome:

- [OpenTherm thermostat with ESPHome and Home Assistant](https://olegtarasov.me/opentherm-thermostat-esphome/)
