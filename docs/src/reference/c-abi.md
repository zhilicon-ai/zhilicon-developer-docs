# Edge-runtime C ABI

The Horizon-1 edge runtime exposes a narrow C ABI that the existing C
firmware under [`src/firmware/`](../../../src/firmware/) links against. The
authoritative header is
[`src/edge-runtime/include/zhilicon_edge.h`](../../../src/edge-runtime/include/zhilicon_edge.h).

## Status codes

| Macro                                 | Value  | Meaning                                        |
| ------------------------------------- | ------ | ---------------------------------------------- |
| `ZHI_EDGE_OK`                         | 0      | Success                                        |
| `ZHI_EDGE_WOULD_BLOCK`                | 1      | Caller should yield                            |
| `ZHI_EDGE_INVALID_ARGUMENT`           | 2      | Malformed input                                 |
| `ZHI_EDGE_NOT_FOUND`                  | 3      | Resource does not exist                         |
| `ZHI_EDGE_NOT_ATTESTED`               | 4      | Caller lacks attestation                        |
| `ZHI_EDGE_ATTESTATION_EXPIRED`        | 5      | Proof expired — refresh required                |
| `ZHI_EDGE_ZONE_MISMATCH`              | 6      | Caller zone ≠ device zone                       |
| `ZHI_EDGE_TMR_MISMATCH`               | 7      | Three-way voter split                           |
| `ZHI_EDGE_SEU_IN_FLIGHT`              | 8      | Operation aborted mid-flight                    |
| `ZHI_EDGE_DEADLINE_MISSED`            | 9      | Scheduler deadline missed                       |
| `ZHI_EDGE_OTA_VERIFY_FAILED`          | 10     | OTA signature or hash check failed              |
| `ZHI_EDGE_OTA_REVISION_UNSUPPORTED`   | 11     | OTA targets unsupported silicon revision        |
| `ZHI_EDGE_HAL_FAULT`                  | 12     | HAL reports hardware fault                      |
| `ZHI_EDGE_CAPACITY_EXCEEDED`          | 13     | Fixed-capacity bound exceeded                   |
| `ZHI_EDGE_INTERNAL`                   | 255    | Internal invariant violated (bug)                |

## Functions

### `const char *zhi_edge_version(void)`

Returns a NUL-terminated version string, valid for the program lifetime.

### `zhi_edge_status_t zhi_edge_sovereign_require(const uint8_t *zone_ptr, size_t zone_len, uint64_t now_us)`

Gate an attested operation. Returns `ZHI_EDGE_OK` if the caller's zone tag
matches the installed sovereign context at `now_us`, else one of the error
codes above.

### `uint64_t zhi_edge_monotonic_us(void)`

Monotonic microsecond clock. Stable across SEU events.

### `uint32_t zhi_edge_tid_krad_x10(void)`

Total ionising dose in krad(Si) × 10.

## Linking

Build the staticlib:

```bash
cd src/edge-runtime
make staticlib          # → target/armv7r-none-eabihf/release/libzhilicon_edge_runtime.a
```

Link it into firmware:

```bash
arm-none-eabi-gcc -I src/edge-runtime/include \
                  src/firmware/… \
                  -L src/edge-runtime/target/armv7r-none-eabihf/release \
                  -lzhilicon_edge_runtime \
                  -o zhilicon-firmware.elf
```

## ABI stability

The Rust `EdgeError` → status-code mapping is **append-only**; we never
renumber an existing code. A major SDK version bump is required before any
breaking change to this header.
