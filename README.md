# homeassistant-petsafe

[![hacs_badge](https://img.shields.io/badge/HACS-Default-41BDF5.svg?style=for-the-badge)](https://github.com/hacs/integration)
<a href="https://www.buymeacoffee.com/dcmeglio" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/default-orange.png" alt="Buy Me A Coffee" height="28" width="119"></a>

PetSafe Integration for Home Assistant

Integrate your PetSafe Smartfeed feeders and Scoopfree litter boxes into Home Assistant

## Contributors
Thank you to these contributors who helped make this component even better!

* @cmccambridge
* @hanscats
* @ireneybean
* @seganku
---

## Fork notes (kriszentner)

This fork exists to run a patched build on a personal Home Assistant instance. It is
`wesleykirkland/homeassistant-petsafe` (which fixed the stale `last_feeding` sensor) plus a fix
for a recursive coordinator refresh loop.

**Changes on top of wesleykirkland/master:**

- **Recursive refresh loop fixed.** `_handle_coordinator_update()` and `async_update()` called
  each other, so the integration hit the PetSafe API every ~10s regardless of `update_interval`
  or the "Enable polling for updates" toggle. `REQUEST_REFRESH_DEFAULT_COOLDOWN` (10s) was the
  only thing setting the rate. Measured on 2 feeders: 66 API fetches per 12 minutes before,
  2 after. Offered upstream as
  [wesleykirkland/homeassistant-petsafe#1](https://github.com/wesleykirkland/homeassistant-petsafe/pull/1),
  and see [dcmeglio/homeassistant-petsafe#22](https://github.com/dcmeglio/homeassistant-petsafe/issues/22).
- **Poll intervals raised to 5 minutes.** `update_interval`, plus `SCAN_INTERVAL` on the sensor,
  switch, button, and select platforms. The last three had none, so they used HA's 30s default.
  The petsafe library README warns that PetSafe can lock an account polled more often than once
  per 5 minutes.

**Do not "simplify" the forced update in `_handle_coordinator_update()`.** `BaseCoordinatorEntity`
declares `should_poll` as a `cached_property` returning False, so the `_attr_should_poll = True`
set in `__init__` is ignored and HA never polls these entities. That forced update is the only
caller of `async_update()`. Replacing it with `async_write_ha_state()` breaks the loop but leaves
`last_feeding` and `next_feeding` permanently `unknown`. The recursion is cut in `async_update()`
instead.

**Known unfixed upstream bug:** `_get_next_feeding_time()` raises `IndexError` when a feeder has
no schedules configured
([dcmeglio/homeassistant-petsafe#27](https://github.com/dcmeglio/homeassistant-petsafe/issues/27)).

**Service-call note:** `device_id` must be a list. `get_feeders_by_device_id()` iterates it, so a
bare string is iterated character by character and the call fails with HTTP 500.

```jsonc
{"device_id": ["4203ba2c..."], "time": "12:00:00"}   // correct
{"device_id": "4203ba2c...",   "time": "12:00:00"}   // HTTP 500
```
