# Cephadm OSD Deployment: Reef to Squid Changes

## Summary

The squid release introduces significant improvements to cephadm's OSD deployment, most notably a **granular device replacement system**, **TPM2 encryption support**, **a new `CephVolume` abstraction layer**, and **daemon deployment refactoring**. Below is a detailed breakdown.

---

## 1. Granular Device Replacement (`replace_block`, `replace_db`, `replace_wal`)

**Files changed:**
- `src/pybind/mgr/cephadm/services/osd.py`
- `src/pybind/mgr/cephadm/module.py`

### What changed

In reef, the `OSD` removal class had a single boolean `replace` flag. In squid, this is expanded into four flags:

| Flag | Purpose |
|------|---------|
| `replace` | Original full-OSD replacement (preserved for backward compat) |
| `replace_block` | Replace only the block (data) device |
| `replace_db` | Replace only the DB device |
| `replace_wal` | Replace only the WAL device |

A new `any_replace_params` property was added to the `OSD` class:

```python
@property
def any_replace_params(self) -> bool:
    return any([self.replace, self.replace_block,
                self.replace_db, self.replace_wal])
```

The draining logic (`start_draining()`, `stop_draining()`) and removal queue (`process_removal_queue()`) now use `any_replace_params` instead of just `self.replace`.

The `zap_osd()` method in `RemoveUtil` now passes device-type-specific flags (`--replace-block`, `--replace-db`, `--replace-wal`) to `ceph-volume lvm zap`.

### Impact on OSD deployment

After device replacement, the device cache is explicitly invalidated (`self.mgr.cache.invalidate_host_devices(osd.hostname)`), and the zap flag is automatically set for replacement operations. This enables automatic reprovisioning of the replacement device.

---

## 2. New `CephVolume` Abstraction Layer

**New file:** `src/pybind/mgr/cephadm/ceph_volume.py` (430 lines)

This is an entirely new module in squid that provides a structured wrapper around `ceph-volume` operations:

- **`CephVolume`** - Base class providing `run()` and `run_json()` for executing ceph-volume commands on hosts
- **`CephVolumeLvmList`** - Subclass that parses `ceph-volume lvm list --format json` output and provides:
  - `block_devices()`, `db_devices()`, `wal_devices()`, `all_devices()` - Device enumeration by type
  - `device_osd_mapping()` - Maps devices to their OSD IDs
  - `is_shared_device()` - Detects if a device is shared across multiple OSDs
  - `is_block_device()`, `is_db_device()`, `is_wal_device()` - Device type checks
  - `clear_replace_header()` - Clears replacement headers on devices

The `CephadmOrchestrator` now instantiates `self.ceph_volume: CephVolume = CephVolume(self)` at startup.

---

## 3. New `replace_device()` Orchestrator Command

**File:** `src/pybind/mgr/cephadm/module.py`

Squid adds a high-level `replace_device()` method that orchestrates device replacement:

1. Runs `ceph-volume lvm list` to identify the device's role and associated OSDs
2. Validates the device exists and checks if it's shared (requires `--yes-i-really-mean-it` for shared devices)
3. Verifies PG safety via `safe_to_destroy()`
4. Automatically determines which replace flags to set (`replace_block`, `replace_db`, `replace_wal`)
5. Enqueues the OSDs for removal with the correct replacement flags

Supports `--clear` to remove replacement headers from a device.

---

## 4. Device Inventory: `being_replaced` Field

**Files changed:**
- `src/python-common/ceph/deployment/inventory.py`
- `src/python-common/ceph/deployment/drive_selection/selector.py`

The `Device` class gains a new `being_replaced: Optional[bool]` attribute. The `DriveSelection` selector now skips devices that are `being_replaced`:

```python
if disk.being_replaced:
    logger.debug('Ignoring disk {} as it is being replaced.'.format(disk.path))
    continue
```

This prevents cephadm from trying to provision new OSDs on a device that is in the process of being replaced.

---

## 5. TPM2 Encryption Support

**Files changed:**
- `src/python-common/ceph/deployment/drive_group.py`
- `src/python-common/ceph/deployment/translate.py`

### DriveGroupSpec

A new `tpm2: bool = False` parameter is added to `DriveGroupSpec` and included in `_supported_features`.

### Ceph-volume translation

When `tpm2` is enabled, the translated ceph-volume command includes `--with-tpm`:

```python
if self.spec.tpm2:
    cmds[i] += " --with-tpm"
```

This enables OSD encryption using TPM2 (Trusted Platform Module) hardware for key management, as an alternative to or in addition to standard dmcrypt.

---

## 6. OSD Daemon Deployment Refactoring

**File:** `src/pybind/mgr/cephadm/services/osd.py`

### `deploy_osd_daemons_for_existing_osds()` signature change

**Reef:**
```python
async def deploy_osd_daemons_for_existing_osds(self, host: str, spec: DriveGroupSpec, ...)
```

**Squid:**
```python
async def deploy_osd_daemons_for_existing_osds(self, host: str, service_name: str, ...)
```

The method no longer requires a full `DriveGroupSpec` object — it only needs the service name string. This decouples OSD daemon deployment from the drive group spec.

### `CephadmDaemonDeploySpec` construction

**Reef** used `self.make_daemon_spec()` which required a full spec and a `network=''` placeholder argument:
```python
daemon_spec = self.make_daemon_spec(
    spec=spec, daemon_id=str(osd_id), host=host,
    daemon_type='osd', network='',
)
```

**Squid** directly constructs `CephadmDaemonDeploySpec`:
```python
daemon_spec = CephadmDaemonDeploySpec(
    service_name=service_name, daemon_id=str(osd_id),
    host=host, daemon_type='osd',
)
```

This removes the unnecessary `network` parameter and simplifies the construction path.

---

## 7. OSD Removal Queue Improvements

**File:** `src/pybind/mgr/cephadm/services/osd.py`

### Return value change

`process_removal_queue()` now returns `bool` (was `None`/void in reef). Returns `True` when an OSD was successfully removed.

### Serve loop integration

In `src/pybind/mgr/cephadm/serve.py`, the serve loop now uses this return value to immediately re-enter the loop when a removal was processed:

```python
removal_queue_result = self.mgr.to_remove_osds.process_removal_queue()
if removal_queue_result:
    continue  # re-enter loop immediately
```

This makes multi-OSD removals faster by not waiting for the full serve cycle between each removal.

### Auto-zap on replacement

When any replacement flag is set, the zap flag is now automatically enabled:
```python
if any_replace_params:
    osd.zap = True
```

---

## 8. Cephadm Daemon Form Refactoring (cephadm client-side)

**New file:** `src/cephadm/cephadmlib/daemons/ceph.py` (482 lines)

In reef, this file did not exist (the OSD daemon form was defined elsewhere in the monolithic cephadm script). Squid refactors the daemon form system with:

- **`Ceph`** base class - A `ContainerDaemonForm` for standard Ceph daemons (mon, mgr, mds, rgw, etc.)
- **`OSD(Ceph)`** subclass - OSD-specific overrides including:
  - `osd_fsid` property for OSD-specific FSID tracking
  - Static sysctl settings (`fs.aio-max-nr`, `kernel.pid_max`)
  - Separate `for_daemon_type()` registration (OSD is special-cased)
- **`CephExporter`** class - Separated from the main Ceph class
- **`get_ceph_mounts_for_type()`** - Device mount logic for OSD containers (e.g., `/dev`, `/sys`, `/run/lvm`, SELinux folder handling)

---

## 9. Init Containers Support

**Files changed:**
- `src/pybind/mgr/cephadm/services/cephadmservice.py`
- `src/pybind/mgr/cephadm/serve.py`

`CephadmDaemonDeploySpec` gains an `init_containers` parameter. Init containers are short-lived containers that run before the main daemon container starts. While this is a general feature, it applies to all daemon types including OSDs.

---

## 10. Stray Daemon Protection

**File:** `src/pybind/mgr/cephadm/serve.py`

A new `recently_altered_daemons` dict tracks recently created/removed daemons with timestamps. Daemons altered within the last 60 seconds are excluded from stray daemon warnings. This prevents false stray-daemon alerts during OSD creation/removal operations.

---

## 11. Memory Autotuning Improvements

**File:** `src/pybind/mgr/cephadm/serve.py`

Enhanced debug logging for OSD memory autotuning:
- Logs total memory, target ratio when autotuning
- Logs when removing `osd_memory_target` for hosts with no OSDs or invalid calculations

---

## File Change Summary (OSD-related only)

| File | Lines Added | Lines Removed | Nature |
|------|:-----------:|:-------------:|--------|
| `ceph_volume.py` (new) | +430 | 0 | New CephVolume abstraction |
| `services/osd.py` | +59 | -12 | Device replacement, queue improvements |
| `module.py` | +150 | -50 | replace_device(), remove_osds() flags |
| `serve.py` | +80 | -15 | Stray protection, removal loop, init containers |
| `cephadmservice.py` | +50 | -20 | Init containers, simplified keyring |
| `daemons/ceph.py` (new) | +482 | 0 | Refactored OSD daemon form |
| `drive_group.py` | +5 | -1 | TPM2 support |
| `translate.py` | +3 | 0 | TPM2 ceph-volume flag |
| `inventory.py` | +7 | -3 | `being_replaced` field |
| `drive_selection/selector.py` | +4 | 0 | Skip replaced devices |
| `test_replace_device.py` (new) | +53 | 0 | Device replacement tests |
| `test_ceph_volume.py` (new) | +231 | 0 | CephVolume tests |
| `ceph_volume_data.py` (new) | +1 | 0 | Test data |
