# stream_server (vendored)

Vendored copy of [oxan/esphome-stream-server](https://github.com/oxan/esphome-stream-server)
— a serial-to-WiFi bridge component for ESPHome.

**Copyright (C) 2020-2023 Oxan van Leeuwen.** Licensed under the GNU General
Public License v3 or later; see `LICENSE.txt`. Because this project
redistributes a modified copy, the project as a whole is GPL-3.0-or-later.

## Why vendored instead of pulled via `external_components`

Upstream does not build on ESPHome 2026.x: `dump_config()` calls
`esphome::network::get_use_address()`, which that release removed.

`stream_server.cpp` is the only modified file — the call is log-only and was
dropped. See `0001-esphome-2026-compat.patch`. Every other file is byte-identical
to upstream master.

Once upstream supports ESPHome 2026.x, drop this directory and switch the YAML
back to:

```yaml
external_components:
  - source: github://oxan/esphome-stream-server
```
