# Local patches (Homewind workspace copy)

This repository is NimBLE-Arduino **2.5.2** (official tag, first commit) plus the
patch below. `library.properties` says `2.5.2+hw1` so the patched copy is visible
in build logs (`Using library NimBLE-Arduino at version 2.5.2+hw1`).

## hw1 — connection handle used as array index (host)

**Problem.** NimBLE-Arduino 2.3.0–2.5.2 keeps per-connection state in global
arrays of `BLE_MAX_CONNECTIONS + 1` entries, indexed with the raw HCI connection
handle:

- `int slave_conn[]` (`ble_gap.c`, also declared `extern` in `ble_hs_hci_evt.c`)
- `g_max_tx_time[]`, `g_max_rx_time[]`, `g_max_tx_octets[]`, `g_max_rx_octets[]` (`ble_gap.c`)

The ESP32-S3 controller hands out handles up to `ble_max_act - 1`, which can be
larger than `BLE_MAX_CONNECTIONS`. With `BLE_MAX_CONNECTIONS = 3`, a peripheral
link (a central such as Zwift connecting to us) got handle 4, and
`slave_conn[4] = 1` overwrote the next global, the list head
`ble_gap_update_entries`. The device then crashed in
`ble_gap_update_entry_remove()` when that link disconnected (Homewind 1.6.0
hardware test, core dump, reproduced twice by closing Zwift).

**Fix.** The state moves into `struct ble_hs_conn` (`ble_hs_conn_priv.h`), as in
upstream esp-nimble (`nimble-1.5.0-idf` and `nimble-1.6.0-idf`, October 2025), where `slave_conn` is a bit in
the connection object; NimBLE-Arduino master (3.0.0-dev) no longer has the array.

- `slave_conn:1`: set under `ble_hs_lock()` after the connect event in
  `ble_gap_rx_rd_rem_sup_feat_complete()`; read in
  `ble_hs_hci_evt_disconn_complete()` (the connection still exists there) and,
  under the lock, in `ble_gap_conn_broken()` before `ble_hs_atomic_conn_delete()`.
- `bhc_max_tx_octets/rx_octets/tx_time/rx_time`: written in
  `ble_gap_rx_data_len_change()`, read in `ble_gap_set_data_len()`, both under
  the lock. A new connection starts at 0 (`ble_hs_conn_alloc()` clears it); for
  an unknown handle `ble_gap_set_data_len()` always sends the command to the
  controller (as upstream; 2.5.2 answered a zero request for an unknown handle
  0..3 with a synthetic event).

Not changed: `host/mesh/src/proxy_msg.c` also indexes `roles[]` with the handle;
mesh is not built in this configuration (`BLE_MESH_PROXY` off).
