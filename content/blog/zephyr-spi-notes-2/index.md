---
title: 🧩 Zephyr SPI Notes for Nordic microcontrollers - Part 2
summary: Practical notes for using SPI on Nordic nRF52 with Zephyr, especially with multiple devices on one bus and Nordic nRF-SPIM.
date: 2026-07-13
authors:
  - me
tags:
  - Zephyr RTOS
  - Nordic microcontrollers
  - Communication protocols
image:
  caption: 'Image credit: [**Me**]'
---

*Practical notes for using SPI on Nordic nRF52 / nRF52840 with Zephyr, especially with multiple devices on one bus and `compatible = "nordic,nrf-spim"`.*

In the first part of these notes, the focus was on getting the SPI hardware description right: selecting the Nordic SPIM controller, assigning pins through `pinctrl`, defining chip-selects, enabling the right Kconfig options, and describing SPI child nodes in devicetree.

That configuration work is essential, but it is only the first half of the story.

Once the SPI bus and its devices are described in devicetree, the C code still has to access them correctly. This is where Zephyr’s devicetree macros, SPI helper APIs, and device model can become confusing, especially when a project mixes manual SPI transactions with driver-backed devices.

This second part focuses on that boundary.

It explains when to use `SPI_DT_SPEC_GET()`, when `SPI_DT_SPEC_INST_GET()` makes sense, and why `DEVICE_DT_GET()` is not a generic way to access any SPI child node. It also explains the common `__device_dts_ord_*` linker error, which usually appears when the code asks for a Zephyr device object that no compiled driver created.

Finally, this post closes with small SPI transfer helpers for manual bring-up: write-only transfers, full-duplex transfers, command-plus-response transactions, RAM-backed buffers, and the separation between generic SPI helpers and device-specific command wrappers.

The goal is not to build a complete Zephyr driver yet. The goal is to make the first application-level SPI transactions predictable, readable, and easier to debug.

---

## 5. Access patterns in C

This section covers two access patterns: manual SPI access and driver-backed device access. It introduces them through three related macros.

```
Manual SPI access:
- SPI_DT_SPEC_GET()
- SPI_DT_SPEC_INST_GET()

Driver-backed device access:
- DEVICE_DT_GET()
```

These access patterns belong to Zephyr's devicetree, SPI and device-model APIs.

Both `SPI_DT_SPEC_GET` and `SPI_DT_SPEC_INST_GET` initialize the same type for manual SPI access:

```c
struct spi_dt_spec
```

while `DEVICE_DT_GET` obtains a pointer to a created driver, providing driver-backed device access:

```c
struct device
```

These macros help with static configuration, portability and compile-time generation. However, they are configuration helpers, not memory-safety guarantees.

### `SPI_DT_SPEC_GET()`

When manually implementing a peripheral's command/register protocol, this is usually the correct choice. It provides the robust `struct spi_dt_spec` initialization as the instance-based macro. More precisely, it accepts a **devicetree node identifier**.

Macros such as this make it possible to use:

```c
DT_NODELABEL(afe)
DT_ALIAS(my_afe)
DT_PATH(soc, spi_4002f000, afe_0)
```

Zephyr distinguishes the macros used to obtain a node identifier from the API that consumes it.

By using this macro, this line:

```c
static const struct spi_dt_spec spi_afe = SPI_DT_SPEC_GET(SPI_NODE_AFE, SPI_OP);
```

collects the SPI information for the child node:

1. Which SPI bus the child belongs to
  
2. Which chip-select index it uses
  
3. Which chip-select configuration corresponds to that index, including the GPIO when chip-select is GPIO-controlled.
  
4. The child device frequency from `spi-max-frequency`
  
5. The SPI operation flags from `SPI_OP`
  

*Example:*

```c
#include <zephyr/device.h>
#include <zephyr/drivers/spi.h>
#include <zephyr/drivers/pinctrl.h>
#include <zephyr/kernel.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(app, LOG_LEVEL_INF);

#define SPI_NODE_ACC DT_NODELABEL(acc)
#define SPI_NODE_FLASH DT_NODELABEL(flash)
#define SPI_NODE_AFE DT_NODELABEL(afe)
#define SPIOP (SPI_WORD_SET(8) | SPI_TRANSFER_MSB | SPI_OP_MODE_MASTER)

static const struct spi_dt_spec spi_acc = SPI_DT_SPEC_GET(SPI_NODE_ACC, SPIOP);
static const struct spi_dt_spec spi_flash = SPI_DT_SPEC_GET(SPI_NODE_FLASH, SPIOP);
static const struct spi_dt_spec spi_afe = SPI_DT_SPEC_GET(SPI_NODE_AFE, SPIOP);

int main(void)
{
    if (!device_is_ready_dt(&spi_acc) || !device_is_ready_dt(&spi_flash)
         || !device_is_ready_dt(&spi_afe))
    {
        LOG_ERR("One or more SPI access paths are not ready");
        return -ENODEV;
    }

    LOG_INF("SPI buses and chip-select controllers are ready");

    while (1) {
        k_sleep(K_SECONDS(1));
    }
}
```

Historically, the SPI acquisition macros accepted three parameters, with the final argument representing a chip-select timing delay in microseconds. In [Zephyr 4.3 and newer](https://github.com/zephyrproject-rtos/zephyr/blob/main/doc/releases/migration-guide-4.3.rst), providing a third argument to `SPI_DT_SPEC_GET` or `SPI_DT_SPEC_INST_GET` is deprecated and triggers build errors or compile-time warnings depending on the compiler configuration.

For the devicetree-based SPI helper macros, chip-select setup and hold timing now belong in devicetree rather than in the macro arguments. These timing figures should be specified through the standardized devicetree properties `spi-cs-setup-delay-ns` and `spi-cs-hold-delay-ns`.

### `SPI_DT_SPEC_INST_GET()`

`SPI_DT_SPEC_INST_GET()` initializes an SPI specification for a numbered instance of `DT_DRV_COMPAT`. Drivers commonly combine it with `DT_INST_FOREACH_STATUS_OKAY()` and `DEVICE_DT_INST_DEFINE()` to create one driver-backed device for every enabled compatible node.

The preprocessor translates these abstractions during early compilation phases:

1. ***Resolution of instance to node:*** The preprocessor evaluates `SPI_DT_SPEC_INST_GET(inst, ...)` by replacing the instance index with a call to `DT_DRV_INST(inst)`
  
2. ***Expression to node identifier:*** `DT_DRV_INST(inst)` expands into the internal devicetree node path token.
  
3. ***Structure initialization***: This token is passed to `SPI_DT_SPEC_GET`, which queries the generated devicetree header files to output a static initializer for a `struct spi_dt_spec`
  

To enable instance-based macros within a driver, the compilation unit must define the compatible token corresponding to the binding’s compatible string.

```c
#define DT_DRV_COMPAT manufacturer_reference
```

*Example:*

[PMW3610 optical mouse sensor driver](https://github.com/badjeff/zmk-pmw3610-driver), updated to Zephyr 4.3:

```c
#define DT_DRV_COMPAT cormoran_pmw3610

#define PMW3610_DEFINE(n)                                                                          \
    static struct pixart_data data##n;                                                             \
    static const struct pixart_config config##n = {                                                \
        .spi = SPI_DT_SPEC_INST_GET(n, PMW3610_SPI_MODE),                                       \
        .irq_gpio = GPIO_DT_SPEC_INST_GET(n, irq_gpios),                                           \
        .cpi = DT_PROP(DT_DRV_INST(n), cpi),                                                       \
        .swap_xy = DT_PROP(DT_DRV_INST(n), swap_xy),                                               \
        .inv_x = DT_PROP(DT_DRV_INST(n), invert_x),                                                \
        .inv_y = DT_PROP(DT_DRV_INST(n), invert_y),                                                \
        .evt_type = DT_PROP(DT_DRV_INST(n), evt_type),                                             \
        .x_input_code = DT_PROP(DT_DRV_INST(n), x_input_code),                                     \
        .y_input_code = DT_PROP(DT_DRV_INST(n), y_input_code),                                     \
        .force_awake = DT_PROP(DT_DRV_INST(n), force_awake),                                       \
        .force_awake_4ms_mode = DT_PROP(DT_DRV_INST(n), force_awake_4ms_mode),                     \
    };                                                                                             \
    DEVICE_DT_INST_DEFINE(n, pmw3610_init, NULL, &data##n, &config##n, POST_KERNEL,                \
                          CONFIG_INPUT_PMW3610_INIT_PRIORITY, &pmw3610_driver_api);

    DT_INST_FOREACH_STATUS_OKAY(PMW3610_DEFINE)
```

### `DEVICE_DT_GET()`

This macro is used when code needs a reference to a driver-created Zephyr device object. It expands to a pointer to the device object created for that node.

The device may not have initialized successfully, which is why you subsequently call:

```c
device_is_ready(dev)
```

Then, the application can use functions such as `sensor_sample_fetch()` or `flash_read()`, through the corresponding subsystem API.

Using the SPI controller directly is also possible:

```c
const struct device *bus = DEVICE_DT_GET(DT_NODELABEL(spi3));
```

However, obtaining the controller pointer is only part of the manual SPI API. The caller must also construct and pass a `struct spi_config` to functions such as:

```c
spi_transceive(bus, &config, &tx, &rx);
```

`SPI_DT_SPEC_GET()` is usually more convenient because it packages the bus pointer and the child-specific frequency, chip-select, and operation configuration into one `struct spi_dt_spec`.

*Example using a [BME280 sensor](https://github.com/circuitdojo/nrf9160-feather-examples-and-drivers/blob/04972ced2ae63dbfa4712f94ca133e4a45a5a71a/samples/bme280/src/main.c#L18)*

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/devicetree.h>
#include <zephyr/drivers/sensor.h>

#define BME280 DT_INST(0, bosch_bme280)
// Note that DT_INST is less explicit than DT_ALIAS or DT_NODELABEL
// Instance numbering is based on compatible instances rather
// than the physical SPI chip-select number

#if !DT_NODE_HAS_STATUS(BME280, okay)
#error Your devicetree has no enabled nodes with compatible "bosch,bme280"
#endif

const struct device *dev = DEVICE_DT_GET(BME280);

int main(void){
     if (!device_is_ready(dev)){
         printk("No device \"%s\" found; did initialization fail?\n",
                 dev->name);
         return -ENODEV;
     }
     else{
     printk("Found device \"%s\"\n", dev->name);
     }
}
```

### Definition of a hardware binding

A binding is required; a driver is optional. A matching binding doesn't create a `struct device` by itself. A driver must instantiate the node with a device-definition macro such as `DEVICE_DT_INST_DEFINE()`.

The current Zephyr device-model documentation confirms that `DEVICE_DT_GET()` only works when a device object was allocated for the node; otherwise, the reference fails at link time.

A small binding for the real device is the clean solution:

```yaml
# dts/bindings/sensor/manufacturer,device.yaml

description: |
  Optical analog front end accessed through application-managed SPI transactions

compatible: "manufacturer,reference"

include: [sensor-device.yaml, spi-device.yaml]

properties:
  reg:
    required: true
  spi-max-frequency:
    type: int
    required: true
    description: Max SPI clock in Hz (up to 80000000)

  # These properties are optional
  int1-gpios:
    type: phandle-array
    required: false
    description: Optional interrupt GPIO connected to the sensor’s INT1 pin

  reset-gpios:
    type: phandle-array
    required: false
    description: Optional reset GPIO
```

Bindings validate nodes and generate the property macros consumed by the C devicetree API.

For a binding and manual application-level SPI access, you'll usually need to modify:

```
app.overlay (if devicetree won't be modified)
dts/bindings/sensor/manufacturer,device.yaml
CMakeLists.txt
src/main.c
prj.conf/defconfig
```

When you add a complete driver, you'll normally also need to either create or modify:

```
drivers/sensor/device.c
drivers/sensor/CMakeLists.txt
drivers/sensor/Kconfig
```

```c
DEVICE_DT_INST_DEFINE(...)
```

A west manifest becomes relevant when the driver is maintained as a separate external repository or Zephyr module. It's not a normal requirement for an application-local binding. Zephyr modules can expose additional DTS roots through their module metadata.

### SPI Operating modes

| Mode | CPOL | CPHA | DTS Properties |
| --- | --- | --- | --- |
| 0   | 0   | 0   | (default) |
| 1   | 0   | 1   | `spi-cpha` |
| 2   | 1   | 0   | `spi-cpol` |
| 3   | 1   | 1   | `spi-cpol; spi-cpha` |

For fixed peripheral characteristics, the cleanest approach is usually adding these properties to devicetree:

```yaml
afe: optical-afe@2 {
    spi-cpol;
    spi-cpha;
};
```

#### Operation Flags

Common flags for `spi_config.operation`:

| Flag | Description |
| --- | --- |
| `SPI_OP_MODE_MASTER` | Controller/master mode (default) |
| `SPI_OP_MODE_SLAVE` | Peripheral/slave mode |
| `SPI_MODE_CPOL` | Clock polarity (idle high) |
| `SPI_MODE_CPHA` | Clock phase (sample on second edge) |
| `SPI_WORD_SET(n)` | Word size in bits (typically 8) |
| `SPI_TRANSFER_MSB` | MSB first (default) |
| `SPI_TRANSFER_LSB` | LSB first |
| `SPI_HOLD_ON_CS` | Keep CS active after transaction |
| `SPI_LOCK_ON` | Lock bus for multiple transactions |
| `SPI_CS_ACTIVE_HIGH` | CS is active high (unusual) |

If using `SPI_LOCK_ON` or `SPI_HOLD_ON_CS`, release when done:

```c
spi_release_dt(&spi_dev);
```

`SPI_HOLD_ON_CS` attempts to leave CS asserted after a transfer, and `SPI_LOCK_ON` prevents other callers from acquiring the controller until released. They should be used cautiously on a bus shared by several devices.

A common starting point when working with SPI peripherals is:

```c
#define SPI_OP \    
(SPI_OP_MODE_MASTER | SPI_WORD_SET(8) | SPI_TRANSFER_MSB)
```

This means:

```
SPI_OP_MODE_MASTER -> The nRF52840 initiates the transfer and drives SCK
SPI_WORD_SET(8)    -> 8-bit words
SPI_TRANSFER_MSB   -> most significant bit first
```

If the device requires another SPI mode, add CPOL and/or CPHA flags:

```c
#define SPI_OP_MODE3 \    
(SPI_OP_MODE_MASTER | SPI_WORD_SET(8) | SPI_TRANSFER_MSB | \
 SPI_MODE_CPOL | SPI_MODE_CPHA)
```

Don't assume all SPI devices on the same bus use the same mode. Check each datasheet.

### Different operation flags per device

If needed, give each device its own operation flags:

```c
#define ACC_SPI_OP \
 (SPI_OP_MODE_MASTER | SPI_WORD_SET(8) | SPI_TRANSFER_LSB)

#define FLASH_SPI_OP \
 (SPI_OP_MODE_MASTER | SPI_WORD_SET(8) | SPI_TRANSFER_MSB | \
  SPI_MODE_CPOL | SPI_MODE_CPHA)

#define AFE_SPI_OP \
(SPI_OP_MODE_MASTER | SPI_WORD_SET(8) | SPI_TRANSFER_MSB)

static const struct spi_dt_spec spi_acc = SPI_DT_SPEC_GET(SPI_NODE_ACC, ACC_SPI_OP);
static const struct spi_dt_spec spi_flash = SPI_DT_SPEC_GET(SPI_NODE_FLASH, FLASH_SPI_OP);
static const struct spi_dt_spec spi_afe = SPI_DT_SPEC_GET(SPI_NODE_AFE, AFE_SPI_OP);
```

This keeps the code ready for devices that later need different CPOL, CPHA, word size, or transfer behavior.

Manually adding CPOL and CPHA flags in C is valid, but the code should follow one primary convention and use it consistently.

---

## 6. `__device_dts_ord_*`: what the error really means

A linker error like this:

```
undefined reference to `__device_dts_ord_...'
```

doesn't always mean that the devicetree node is missing.

More precisely, it means:

```
The C code asked Zephyr for a device object,
but no compiled driver created that device object.
```

A common trigger is, for example:

```c
const struct device *dev = DEVICE_DT_GET(DT_NODELABEL(radio1));
```

This only works if `radio1` is a real Zephyr device instance, backed by:

```
- a valid devicetree node
- status = "okay"
- a compatible string with a matching binding
- a compiled driver
- and the Kconfig symbols needed to include that driver
```

If any of those pieces are inconsistent, the devicetree node may appear in `zephyr.dts`, but the linker can still fail with `__device_dts_ord_*`.

For solving this issue, there are two different questions to answer:

```
1. Does the node exist in devicetree?
2. Did a driver create a Zephyr struct device for that node?
```

`DT_NODELABEL(radio1)` can be valid while `DEVICE_DT_GET(DT_NODELABEL(radio1))` still fails.

### Why this error happens

In SPI bring-ups, this type of issue appears because several configuration layers must agree with each other:

```
- .dts / .dtsi describe the hardware
- bindings define which properties are legal for each compatible
- prj.conf / defconfig select software features and drivers
- C code chooses either DEVICE_DT_GET() or SPI_DT_SPEC_GET()
```

When those layers contradict each other, Zephyr may generate a devicetree that looks plausible, but the final device object is still missing.

### Case 1: Generic SPI child accessed as a real device

This pattern is unsafe for a generic SPI child:

```c
afe: optical-afe@2 {
    compatible = "vnd,spi-device";
    reg = <2>;    
    spi-max-frequency = <1000000>;
};
```

```c
const struct device *dev = DEVICE_DT_GET(DT_NODELABEL(afe));
```

The node exists, but there isn't a dedicated Zephyr driver creating a `struct device` for it.

For manual SPI transactions, use:

```c
#include <zephyr/drivers/spi.h>

#define AFE_NODE DT_NODELABEL(afe)
#define AFE_SPI_OP \
(SPI_OP_MODE_MASTER | SPI_WORD_SET(8) | SPI_TRANSFER_MSB)

static const struct spi_dt_spec spi_afe = SPI_DT_SPEC_GET(AFE_NODE, AFE_SPI_OP);
```

Then check:

```c
if (!spi_is_ready_dt(&spi_afe)) {
    return -ENODEV;
}
```

In Zephyr versions where `"vnd,spi-device"` works, the matching binding is found under the test bindings:

```yaml
zephyr/dts/bindings/test/vnd,spi-device.yaml
```

That binding includes the common `spi-device.yaml` properties required by macros such as `SPI_DT_SPEC_GET()`.

However, `vnd,spi-device` is not a wildcard compatible and doesn't mean “any SPI peripheral”. Devicetree compatibles are matched against bindings by their exact strings. Here, `vnd` is only a placeholder vendor prefix and the matching binding exists for Zephyr’s internal tests, not as a production description of arbitrary SPI hardware.

#### Why `SPI_DT_SPEC_GET()` is different

`SPI_DT_SPEC_GET()` doesn't require the SPI child to have its own full driver.

It creates an SPI access descriptor from the child node, using the hardware description provided by the binding:

```
- bus device
- SPI frequency
- chip-select information
- SPI operation flags
```

However, the SPI bus itself still needs a real driver.

So this can work:

```c
static const struct spi_dt_spec spi_afe = SPI_DT_SPEC_GET(DT_NODELABEL(afe), AFE_SPI_OP);
```

because the actual Zephyr device object being required is the SPI controller, such as `&spi3`, not a full dedicated driver for the SPI child.

### Case 2: The SPI bus or GPIO controllers weren't fully enabled

Another issue can be solved by explicitly enabling the GPIO-related nodes in devicetree:

```yaml
&gpio0 {
    status = "okay";
};

&gpio1 {
    status = "okay";
};

&gpiote {
    status = "okay";
};
```

This matters because some SPI children use GPIO chip-selects, interrupt pins, or both.

The SPI bus may look correct, but chip-select and interrupt handling can still depend on the GPIO controller nodes being available.

For SPI with `cs-gpios`, this means the hardware description must include both the SPI controller configuration:

```yaml
&spi3 {
    status = "okay";
    compatible = "nordic,nrf-spim";
    cs-gpios = <&gpio0 17 GPIO_ACTIVE_LOW>,
    <&gpio0 15 GPIO_ACTIVE_LOW>, 
    <&gpio0 13 GPIO_ACTIVE_LOW>;
};
```

and the GPIO controller configuration:

```yaml
&gpio0 {
    status = "okay";
};

&gpio1 {
    status = "okay";
};
```

If a chip-select pin references `&gpio0` or `&gpio1`, that GPIO controller must be usable.

### Case 3: Forcing the wrong Kconfig symbols

One project trap is trying to force low-level nrfx symbols directly, especially:

```
CONFIG_NRFX_GPIOTE=y
```

That's the wrong level of control. It's better to:

```
Enable the Zephyr feature you need.
Let Zephyr/NCS select the low-level nrfx backend when possible.
```

For SPI bring-up, start with:

```
CONFIG_SPI=y
CONFIG_GPIO=y
CONFIG_PINCTRL=y
CONFIG_LOG=y
```

If GPIO interrupts are needed later, enable the Zephyr-facing GPIO and interrupt features, then describe the interrupt pin in devicetree or in the custom binding. Don't blindly force internal nrfx symbols.

---

## 7. Basic SPI transmit / receive helpers

After creating a `struct spi_dt_spec` with `SPI_DT_SPEC_GET()`, the next step is to perform simple SPI transfers.

For manual bring-up, keep the helpers small and predictable:

```
- One helper for write-only transfers
- One helper for full-duplex transfers
- One helper for command + response in one CS window
```

These helpers aren't device drivers. They only move bytes over SPI.

Device-specific details such as register headers, opcodes, read bits, dummy bytes, and byte order should be handled by device-specific wrapper functions.

Dummy bytes depend on the peripheral. Some devices return useful data during the same bytes used to send a command. Others ignore the first received byte and return valid data only after the command or register address has been shifted in. Some devices also require one or more extra clock cycles before the response becomes valid.

When dummy bytes are needed, they aren't meaningful commands. They are bytes sent on MOSI so that the SPI controller continues generating SCK pulses while the peripheral shifts out the response on MISO.

### Write-only transaction

Use this when the peripheral only needs bytes on MOSI and the MISO response is irrelevant.

```c
static int spi_write_bytes(const struct spi_dt_spec *spi,
                           const uint8_t *tx_data,
                           size_t tx_len)
{
    const struct spi_buf tx_buf = {
        .buf = (void *)tx_data,
        .len = tx_len,
    };

    const struct spi_buf_set tx = {
        .buffers = &tx_buf,
        .count = 1,
    };

    return spi_write_dt(spi, &tx);
}
```

Typical use:

```c
// Example: write-enable command for some flash devices
uint8_t cmd[] = { 0x06 }; 

ret = spi_write_bytes(&spi_flash, cmd, sizeof(cmd)); 

if (ret != 0) {\
    LOG_ERR("SPI write failed: %d", ret); 
}
```

### Full-duplex transceive

SPI is normally full-duplex. Every byte clocked out on MOSI also clocks one byte in on MISO.

Use this helper when the transmit and receive lengths are the same:

```c
static int spi_transceive_bytes(const struct spi_dt_spec *spi,
                                const uint8_t *tx_data,
                                uint8_t *rx_data,
                                size_t len)
{
    const struct spi_buf tx_buf = {
        .buf = (void *)tx_data,
        .len = len,
    };

    struct spi_buf rx_buf = {
        .buf = rx_data,
        .len = len,
    };

    const struct spi_buf_set tx = {
        .buffers = &tx_buf,
        .count = 1,
    };

    const struct spi_buf_set rx = {
        .buffers = &rx_buf,
        .count = 1,
    };

    return spi_transceive_dt(spi, &tx, &rx);
}
```

*Example:*

```c
uint8_t tx[2] = { 0x00, 0x00 }; 
uint8_t rx[2] = { 0 }; 

ret = spi_transceive_bytes(&spi_acc, tx, rx, sizeof(tx)); 
if (ret != 0) { 
    LOG_ERR("SPI transceive failed: %d", ret);
} 
```

SPI is electrically full-duplex, but Zephyr lets the software describe different amounts of data to transmit and receive.

`spi_transceive_dt()` is just the devicetree wrapper around:

```c
spi_transceive(spec->bus, &spec->config, tx_bufs, rx_bufs)
```

The API accepts independent TX and RX buffer sets, and either one may even be `NULL` if that direction isn't used.

For some peripherals, extra transmitted bytes can safely act as dummy bytes while the device shifts out a response. For others, extra bytes may be interpreted as additional commands or data. The datasheet should define which transaction pattern is valid.

### Write command, then read response in one CS window

Many SPI devices require this shape:

```
CS low
send command/header bytes
send dummy bytes to generate additional clock cycles (if any)
read response bytes on MISO
CS high
```

This must happen as **one SPI transaction** if the device expects CS to remain asserted.

```c
static int spi_cmd_read(const struct spi_dt_spec *spi,
                        const uint8_t *cmd,
                        size_t cmd_len,
                        uint8_t *response,
                        size_t response_len)
{
    uint8_t tx_buf_storage[32] = {0};
    uint8_t rx_buf_storage[32] = {0};

    if ((cmd_len + response_len) > sizeof(tx_buf_storage)) {
        return -EINVAL;
    }

    memcpy(tx_buf_storage, cmd, cmd_len);

    const size_t total_len = cmd_len + response_len;

    const struct spi_buf tx_buf = {
        .buf = tx_buf_storage,
        .len = total_len,
    };

    struct spi_buf rx_buf = {
        .buf = rx_buf_storage,
        .len = total_len,
    };

    const struct spi_buf_set tx = {
        .buffers = &tx_buf,
        .count = 1,
    };

    const struct spi_buf_set rx = {
        .buffers = &rx_buf,
        .count = 1,
    };

    int ret = spi_transceive_dt(spi, &tx, &rx);
    if (ret != 0) {
        return ret;
    }

    memcpy(response, &rx_buf_storage[cmd_len], response_len);
    return 0;
}
```

### RAM-backed buffers are the safest bring-up pattern

For nRF52840 with `nordic,nrf-spim`, the SPI master uses EasyDMA. Zephyr’s Nordic SPIM binding describes `nordic,nrf-spim` as the Nordic SPI master with EasyDMA.

During bring-up, prefer RAM-backed buffers:

```c
uint8_t tx_buf[8] = {0};
uint8_t rx_buf[8] = {0};
```

or:

```c
static uint8_t tx_buf[32];
static uint8_t rx_buf[32];
```

During bring-up, prefer simple RAM-backed buffers whose lifetime is obvious. Stack buffers are fine for synchronous SPI calls, and static buffers are also safe. Avoid making the first SPI tests depend on pointers to temporary data, returned local arrays, heap allocations, or constant data that may be placed in flash instead of RAM.

### Device-specific wrappers should build the command

The generic helper shouldn't know what a specific device command means.

Better structure:

```
generic SPI helper:  
    "send these bytes and receive these bytes"

device wrapper: 
    "build the command format required by this chip"
```

For example:

```c
static int afe_read_raw(const struct spi_dt_spec *spi, 
                             const uint8_t *cmd,
                             size_t cmd_len,
                             uint8_t *data,
                             size_t data_len){
    return spi_cmd_read(spi, cmd, cmd_len, data, data_len);
}
```

Then the AFE-specific code is responsible for building the correct command/header bytes.

This matters because some devices don't just require a 16-bit register value. The SPI transaction may need to include device-specific command/header bytes, register address bytes, and data bytes.

So avoid code that does this:

```c
uint8_t tx[] = {
    value_msb,
    value_lsb,
};
```

when the device actually expects something closer to:

```
command/header bytes + register address bytes + data bytes
```

The exact byte layout belongs in in device-specific wrapper functions, not in the generic SPI helper.

---

### Example: identity read shape

Most bring-up reads have this form:

```c
uint8_t cmd[] = {/* device-specific read command/header bytes */};
uint8_t id[2] = {0};

ret = spi_cmd_read(&spi_afe, cmd, sizeof(cmd), id, sizeof(id));

if (ret != 0) {
    LOG_ERR("AFE ID read failed: %d", ret);
} 
else {
    LOG_INF("AFE ID bytes: 0x%02x 0x%02x", id[0], id[1]);
}
```

The key point is not the placeholder command. The key point is the transaction shape:

```
one CS assertion
command/header first
dummy clocks next, if needed
response copied after command bytes
```

---

### Example: write-register shape

A write-register wrapper may look like this:

```c
static int device_write_register_raw(const struct spi_dt_spec *spi,
                                     const uint8_t *frame,
                                     size_t frame_len){
    return spi_write_bytes(spi, frame, frame_len);
}
```

The caller builds the complete frame:

```c
uint8_t frame[] = {    
    /* command/header bytes */
    /* register address bytes */
    /* data bytes */
};

ret = device_write_register_raw(&spi_afe, frame, sizeof(frame));
```

For complex devices, the frame must include the command/header bytes required by the datasheet.

---

# Further reading

[Introduction to Devicetree Bindings — Zephyr Project Documentation](https://docs.zephyrproject.org/latest/build/dts/bindings-intro.html)

[Troubleshooting devicetree — Zephyr Project Documentation](https://docs.zephyrproject.org/latest/build/dts/troubleshooting.html)

[Modules (External projects) — Zephyr Project Documentation](https://docs.zephyrproject.org/latest/develop/modules.html)

[Application Development — Zephyr Project Documentation](https://docs.zephyrproject.org/latest/develop/application/index.html)