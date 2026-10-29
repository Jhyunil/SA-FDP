# SA-FDP: Flexible Data Placement for the FEMU SSD Emulator

This repository implements NVMe 2.1 **Flexible Data Placement (FDP)** inside the
black-box SSD (`bbssd`) FTL of [FEMU](https://github.com/vtess/FEMU), a QEMU-based,
DRAM-backed NVMe SSD emulator. It covers placement-aware writes, per-Reclaim-Unit
write pointers, garbage collection that reports *Media Reallocated* events, and the
FDP Events log page.

It builds on the FDP-enabled FEMU fork
[khb84341/FDP-Project](https://github.com/khb84341/FDP-Project), which supplies the FDP
data structures and the NVMe controller plumbing (endurance group setup, the FDP
Configurations / RUH Usage / Statistics log pages, and `IO_MGMT_RECV`). The FTL-side
placement logic, GC event logging and the Events log page handler are implemented here.

> This repository contains only the `hw/femu` subtree of FEMU. See
> [Build and run](#build-and-run) for how to drop it into a full FEMU tree.

---

## Contents

1. [FDP in brief](#fdp-in-brief)
2. [Environment and assumptions](#environment-and-assumptions)
3. [Data structures](#data-structures)
4. [Implementation](#implementation)
5. [Changed files](#changed-files)
6. [Build and run](#build-and-run)
7. [Testing](#testing)
8. [Notes and limitations](#notes-and-limitations)

---

## FDP in brief

| Term | Meaning |
|---|---|
| **Reclaim Unit (RU)** | The striping and erase unit inside a Reclaim Group. Here an RU is a superblock made of one block from every LUN in the group. |
| **Reclaim Group (RG)** | A group of physical dies (LUNs). An RG contains one or more RUs. |
| **Placement Identifier (PID)** | A 16-bit value that names an RG and a Placement Handle, and thereby a specific RU. At most 128 PIDs. |
| **Placement Handle (PHNDL)** | A namespace-scoped host stream resource, mapped to an RUH. |
| **Reclaim Unit Handle (RUH)** | An endurance-group-scoped controller stream resource that references the RU currently being written. |
| **RGIF** | Number of bits in the PID that encode the RG ID. Zero when there is a single RG. |
| **RUHT** | RUH type. *Initially Isolated* (0x1) lets GC mix data from different streams in one RU; *Persistently Isolated* (0x2) keeps streams apart across GC. |

An FDP write reuses the previously reserved **DTYPE** and **DSPEC** fields of the NVMe
write command: DTYPE is set to `0x02` (Data Placement directive) and DSPEC carries the
PID (RG + PHNDL). The controller splits the PID into RGID and PHNDL using RGIF.

FDP adds four log pages:

- `0x20` FDP Configurations
- `0x21` Reclaim Unit Handle Usage
- `0x22` FDP Statistics (HBMW / MBMW / MBE)
- `0x23` FDP Events (four host events, two controller events, a 63-entry ring buffer)

The **Media Reallocated** controller event (`0x80`) tells the host which LBAs GC moved
and how many (NLBAM). Generating and exposing that event is the core of this work.

---

## Environment and assumptions

- Guest kernel: Linux **v6.2.0** or newer (FDP writes go through `io_uring_cmd`)
- Tools: **fio** for FDP writes, **xNVMe** for reading the FDP log pages
- NVM subsystem: a **single namespace (NSID 1)** and a **single endurance group (ID 1)**
- The endurance group starts with **four RUHs** and an identity mapping from placement
  handles to RUHs (`ph #0 → ruh #0`, `ph #1 → ruh #1`, …); no namespace management is needed
- Every RUH is **Initially Isolated**
- `RG_DEGREE = 16`: each RG spans 16 LUNs (`nvme.h`)

With the default geometry from `femu-scripts/run-blackbox.sh` (8 channels, 8 LUNs per
channel, 256 pages per block, 256 blocks per plane, 4 KiB pages) this yields:

| Item | Value |
|---|---|
| Total LUNs | 64 |
| Reclaim groups (`nrg`) | 4 |
| RGIF | 2 bits |
| RU size | 16 blocks × 256 pages × 4 KiB = 16 MiB |
| RUs per RG (`tt_rus`) | 256 |
| RUHs | 4 |

---

## Data structures

All of these live in `bbssd/ftl.h`.

```
struct ssd
 ├─ struct ruh     *ruhtbl   // RUH table, one entry per RUH
 │    ├─ int  ruht            // RUH type
 │    └─ int *cur_ruids       // RU currently being written, per RG
 └─ struct ru_mgmt *rums     // RU manager, one per RG
      ├─ struct ru *rus       // RU array (tt_rus entries); each RU owns a write pointer
      ├─ free_ru_list         // free RUs; RUs reclaimed by GC are appended at the tail
      ├─ victim_ru_pq         // priority queue ordered by valid page count
      ├─ full_ru_list         // RUs that are full and have no invalid pages
      ├─ ii_gc_ruid           // RU reserved for Initially Isolated GC
      └─ total_valid_cnt      // valid pages moved by the current GC run (added here)
```

Stock FEMU keeps a single active superblock ("line"). FDP needs one active RU per
RUH × RG combination, so every RU carries its own write pointer
(`ch, lun, pl, blk, pg`).

---

## Implementation

### 1. DTYPE and PID parsing — `ssd_write()` (`bbssd/ftl.c`)

- Extracts **DTYPE** from `CDW12[23:20]` and **DSPEC (PID)** from `CDW13[31:16]` of the write command.
- Splits the PID using the endurance group's RGIF: the **upper RGIF bits are the RGID**,
  the **lower bits are the PHNDL**.
- If DTYPE is not `0x02`, falls back to `rgid = 0`, `ph = 0`. As the spec requires, a
  write with a bad or missing stream ID still succeeds.
- Resolves `ruhid = ns->fdp.phs[ph]`, then runs `do_gc()` on that RG until its free RU
  count is above the high watermark before writing page by page.

### 2. Picking the physical page — `get_new_page()` (`bbssd/ftl.c`)

- Host writes (`for_gc == false`) use the RU in `ruhtbl[ruhid].cur_ruids[rgid]`;
  GC writes (`for_gc == true`) use `rums[rgid].ii_gc_ruid`.
- Returns the selected RU's write pointer as a `struct ppa`.
- `fdp_gc_write_page()` calls the same function, so relocated data always lands in the
  RG's dedicated GC RU (Initially Isolated behaviour).

### 3. Advancing the write pointer — `ssd_advance_ru_write_pointer()` (`bbssd/ftl.c`)

- Stripes in the same **channel → LUN → page** order as stock FEMU, but confines the
  channel range to the RG's own slice
  (`[rgid·RG_DEGREE/luns_per_ch, (rgid+1)·RG_DEGREE/luns_per_ch)`).
- When the last page of the RU is written:
  - `get_next_free_ruid()` allocates a fresh RU, updates `cur_ruids[rgid]` (or
    `ii_gc_ruid` for GC) and records the new RU's `ruhid` so GC can tell which RUH it belonged to.
  - The finished RU goes to `full_ru_list` via `QTAILQ_INSERT_TAIL()` if it has **no
    invalid pages**, otherwise straight into `victim_ru_pq` via `pqueue_insert()`.
- If no free RU is left, an error is logged and the pointer is left unchanged.

### 4. Media Reallocated event logging — `do_gc()` / `clean_one_block()` (`bbssd/ftl.c`)

- Every valid page GC relocates increments `rum->total_valid_cnt` and pushes its LPN
  into a **min-heap** (`vpg_list`, see `create_heap` / `insert_minheap` / `delete_minheap`).
- After all blocks of the victim RU are cleaned, LPNs are popped in ascending order to
  find the **longest run of contiguous LPNs**; its first LPN is converted to a sector LBA.
- If the RUH is Initially Isolated and its event filter enables `FDP_EVT_MEDIA_REALLOC`,
  an entry is taken from the controller event ring via `nvme_fdp_alloc_event()` and filled:

  | Field | Value |
  |---|---|
  | `type` | `0x80` (Media Reallocated) |
  | `flags` (FDPEF) | `0x07` = PIV ∣ NSIDV ∣ LV |
  | `rgid`, `ruhid`, `nsid` | RG / RUH / namespace of the GC run |
  | `pid` | **start LBA** of the longest contiguous run |
  | `timestamp` | **NLBAM** (pages moved × sectors per page) |

  xNVMe does not print the event-specific field, so the LBA and NLBAM are mirrored into
  the PID and Timestamp fields to make them visible in `xnvme log-fdp-events` output.
- The ring holds 63 events and overwrites the oldest one when full (`nvme-io.c`).

### 5. FDP Events log page — `nvme_fdp_events()` (`nvme-admin.c`)

- Returns `NVME_INVALID_FIELD | NVME_DNR` when `endgrpid != 1`.
- Selects the host or controller event buffer from the LSP bit (`CDW10[8]`).
- If the buffer is empty, transfers a zeroed header only.
- Otherwise allocates `NvmeFdpEventsLog` header + `nelems` events, sets `num_events`,
  flattens the ring buffer starting at `start`, and DMAs `min(buf_len, log_size)` bytes
  to the host with `dma_read_prp()` (`off` is always 0).

### Other changes

- `nvme-util.c`: the PRACT check in `femu_nvme_rw_check_req()` is disabled so writes
  that carry DTYPE/DSPEC are not rejected.
- `bbssd/ftl.c` `ftl_thread()`: `NVME_CMD_IO_MGMT_RECV` (RUH status query) completes with zero latency.
- `nvme-io.c` ends with a commented-out nvme-cli script that fills the namespace
  sequentially and then overwrites it randomly to provoke GC.

---

## Changed files

Relative to the base fork (`hw/femu` of `khb84341/FDP-Project`):

| File | Change |
|---|---|
| `bbssd/ftl.c` | Items 1–4, min-heap helpers, GC event logging |
| `bbssd/ftl.h` | `ru_mgmt.total_valid_cnt`, min-heap (`Heaptype`) declarations |
| `nvme-admin.c` | Item 5 (`nvme_fdp_events`) |
| `nvme-io.c` | Event ring bookkeeping, test script comment |
| `nvme-util.c` | PRACT check disabled |
| `bbssd/bb.c` | I/O command handler renamed (behaviour unchanged) |

`NvmeEnduranceGroup` (`nvme.h`) and `nvme_init_endgrps()` (`femu.c`) come from the base
fork and initialise RGIF, NRG, RUNS and the RUH table.

---

## Build and run

Because this repository holds only `hw/femu`, place it inside the base FEMU tree:

```bash
git clone https://github.com/khb84341/FDP-Project.git
cd FDP-Project
rm -rf hw/femu
git clone https://github.com/Jhyunil/SA-FDP.git hw/femu

mkdir -p build-femu && cd build-femu
cp ../femu-scripts/femu-copy-scripts.sh . && ./femu-copy-scripts.sh .
./femu-compile.sh            # ../configure --enable-kvm --target-list=x86_64-softmmu && make
./run-blackbox.sh            # femu_mode=1 (BBSSD); expects $HOME/images/u20s.qcow2
```

Inside the guest, install a 6.2 kernel plus fio and xNVMe:

```bash
sudo apt install linux-image-6.2.0-37-generic linux-headers-6.2.0-37-generic
sudo reboot && uname -r
```

---

## Testing

### FDP writes with fio

`fdp_pli` indexes the placement ID list returned by `IO_MGMT_RECV`, which is laid out as
`fdp_pli = (RUH index × number of RGs) + RG index`. With 2 RGs and 2 RUHs, for example:

| `fdp_pli` | RG | RUH |
|---|---|---|
| 0 | 0 | 0 |
| 1 | 1 | 0 |
| 2 | 0 | 1 |
| 3 | 1 | 1 |

```ini
[global]
ioengine=io_uring_cmd
cmd_type=nvme
filename=/dev/ng0n1
direct=1
bs=4k
rw=randrw
rwmixwrite=70
fdp=1
time_based=1
runtime=120

[stream0]
fdp_pli=0
offset=0
size=2G

[stream1]
fdp_pli=1
offset=2G
size=2G
```

### FDP log pages with xNVMe

```bash
xnvme log-fdp-config /dev/nvme0n1 --lsi 0x01          # RGIF, NRG, NRUH, RUNS, ...
xnvme log-ruhu       /dev/nvme0n1 --lsi 0x01          # RUH attributes (host / controller specified)
xnvme log-fdp-stats  /dev/nvme0n1 --lsi 0x01          # HBMW / MBMW / MBE
xnvme log-fdp-events /dev/nvme0n1 --lsi 0x01 --lsp 0x00 --limit 4   # controller events (Media Reallocated)
```

- `--lsi`: endurance group ID (always 1)
- `--lsp`: `0x00` for controller events, `0x01` for host events
- `--limit`: number of events to print

Once GC has run, `log-fdp-events` should show the **LBA in the PID field** and
**NLBAM in the Timestamp field** for each Media Reallocated event.

### Forcing GC with nvme-cli

The script kept as a comment at the end of `nvme-io.c` uses
`nvme write --dir-type=0x2 --dir-spec=<PID>` to fill the whole namespace sequentially
and then overwrite it at random, which quickly triggers GC and Media Reallocated events.

---

## Notes and limitations

- `NvmeFdpEvent.pid` is a 16-bit field, so an LBA above 65535 is truncated to its low
  bits. The mirroring into PID/Timestamp exists only to make the values visible through
  xNVMe; it is not where the spec places the LBA.
- The LPN min-heap (`vpg_list`) is a single global shared by all RGs. GC runs
  sequentially on the FTL thread, so there is no concurrent access.
- Persistently Isolated RUHs are not handled; every RUH is assumed to be Initially Isolated.

---

## Acknowledgements

- [FEMU](https://github.com/vtess/FEMU) (Li et al., FAST '18) for the emulator.
- [khb84341/FDP-Project](https://github.com/khb84341/FDP-Project) for the FDP-enabled FEMU base.
