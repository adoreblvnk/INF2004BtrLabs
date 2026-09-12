# BUG HUNT #2: Framing & Serialisation

fix 7 defects in UART packet encoding, checksum verification & stream synchronisation across frame codec & Pico link driver.

---

## test on host first

validate codec logic locally on laptop with unit tests & sanitizers:

```sh
# compile & run host test harness
gcc -Wall -Wextra -o bughunt2_host bughunt2_host.c frame.c
./bughunt2_host

# compile with AddressSanitizer & UndefinedBehaviorSanitizer
gcc -Wall -Wextra -fsanitize=address,undefined -g -o bughunt2_check bughunt2_host.c frame.c
./bughunt2_check
```

---

## defects to fix (7 total)

1. line 7 (`frame.c`, inside `frame_encode`): struct padding & memory alignment. The expression `sizeof(reading_t)` evaluates to 12 bytes due to 32-bit alignment padding inserted by the compiler between `temp_c_x10` and `timestamp_ms`. The wire specification requires exactly 9 packed payload bytes. Set payload length to the constant `FRAME_PAYLOAD` (9) & serialize fields explicitly rather than relying on `sizeof`.
2. lines 11-19 (`frame.c`, inside `frame_encode`): host byte order (endianness). Direct memory copying (`(const uint8_t *)r`) preserves host little-endian format (e.g. `0x1234` becomes `34 12`). The protocol specification mandates network big-endian format (MSB first) for multi-byte fields (`sensor_id`, `temp_c_x10`, `timestamp_ms`). Extract each byte using explicit bitwise shifts (`>> 24`, `>> 16`, `>> 8`).
3. line 16 & line 18 (`frame.c`, inside `frame_encode`): short copy loop & incomplete checksum. The loop condition `i < len - 1` copies only 8 bytes instead of all 9 payload bytes, causing a mismatch between the declared length and emitted bytes, while omitting the 9th byte from the checksum sum. Iterate across all `len` payload bytes (`i < len`).
4. line 48 (`frame.c`, inside `frame_decode`): signed integer promotion in left shift. The expression `(payload[5] << 24)` shifts an 8-bit unsigned integer promoted to signed 32-bit `int`. When the MSB is 1 (values $\ge$ 128), shifting into bit 31 invokes undefined behavior in C. Explicitly cast `payload[5]` to `(uint32_t)` before shifting left by 24.
5. lines 34-37 (`frame.c`, inside `frame_decode`): unchecked payload length & stack buffer bounds. The variable `len = in[1]` is read directly from external input without validation. An untrusted length exceeding `FRAME_MAX` overruns the stack buffer `payload[]`, while a truncated `in_len` reads beyond the input buffer. Verify that `len == FRAME_PAYLOAD` & `in_len == len + 3` before copying or decoding.
6. line 40 & line 51 (`bughunt2_pico.c`, inside `link_init` & `send_reading`): text newline translation on binary UART link. Enabling `uart_set_translate_crlf(LINK_UART, true)` & transmitting with `uart_putc()` translates any binary byte matching `0x0A` (ASCII LF, e.g. in timestamps or sensor IDs) into `0x0D 0x0A`, corrupting binary packets. Disable CRLF translation (`false`) & use `uart_putc_raw()` to transmit raw bytes.
7. lines 59-78 (`bughunt2_pico.c`, inside `rx_poll`): rigid receive window & missing frame resynchronisation. The receiver counts 12 raw bytes unconditionally without hunting for the start-of-frame byte (`FRAME_SOF = 0xAA`). A single dropped or spurious noise byte shifts frame alignment permanently. Implement a frame hunter state machine that searches for `0xAA`, verifies payload length, validates checksum, and resynchronises upon error.

---

## build & flash to Pico W

macOS:
```sh
# copy SDK import helper
cp ~/pico/pico-sdk/external/pico_sdk_import.cmake .
# configure & build
mkdir -p build && cd build
cmake -DPICO_BOARD=pico_w ..
make -j8 bughunt2
# flash to Pico W (bootsel mode)
cp bughunt2.uf2 /Volumes/RPI-RP2
# open serial monitor (Ctrl-A Ctrl-\ to exit)
screen /dev/tty.usbmodem* 115200
```

windows (powershell):
```powershell
# copy SDK import helper
Copy-Item C:\pico\pico-sdk\external\pico_sdk_import.cmake .
# configure & build
New-Item -ItemType Directory -Force -Path build
cd build
cmake -DPICO_SDK_PATH="C:\pico\pico-sdk" ..
cmake --build . --target bughunt2
# flash to Pico W
Copy-Item bughunt2.uf2 -Destination D:\
```

---

## expected output

host test harness:
```
BUG HUNT #2 - frame codec
sizeof(reading_t) = 12 bytes

--- vector A: normal ---
expected   [12] AA 09 12 34 01 00 FD 0A 0B 0C 0D 72 
encoded    [12] AA 09 12 34 01 00 FD 0A 0B 0C 0D 72 

--- vector B: high timestamp, negative temp ---
expected   [12] AA 09 00 07 00 FF C9 80 11 22 33 B5 
encoded    [12] AA 09 00 07 00 FF C9 80 11 22 33 B5 

--- vector C: extremes ---
expected   [12] AA 09 FF FF FF 80 00 00 00 00 00 7D 
encoded    [12] AA 09 FF FF FF 80 00 00 00 00 00 7D 

--- malformed frames (must all be rejected) ---
CODEC MATCHES THE SPECIFICATION  (0 failures)
```

Pico serial monitor output (press GP20 to transmit):
```
BUG HUNT #2 - press GP20 to send a reading
sizeof(reading_t) = 12 bytes

tx         [12] AA 09 12 34 01 00 FD 0A 0B 0C 0D 72 
rx         [12] AA 09 12 34 01 00 FD 0A 0B 0C 0D 72 
           id=0x1234 status=1 temp=25.3 C t=168496141 ms
```
