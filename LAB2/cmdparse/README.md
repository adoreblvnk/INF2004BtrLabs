# CHALLENGE: In-Place Command Parser

zero-allocation ASCII command parser for embedded serial interfaces without `malloc` or `strtok`.

---

## embedded constraints & design rules

parsers running on bare-metal microcontrollers operate under strict memory & reentrancy constraints.

- no dynamic allocation (`malloc`): microcontrollers operate with 264 KB SRAM & lack memory management units (MMU). Heap fragmentation leads to silent allocation exhaustion in long-running embedded systems. All state must live on the stack or in pre-allocated buffers.
- no `strtok()`: standard library `strtok()` stores its internal scanning state in a hidden `static` pointer. If 2 UART channels parse concurrently, or if an interrupt service routine preempts parsing, both buffers corrupt silently. Parse delimiters manually without static globals.
- tokenise in place: mutate the caller's buffer directly by replacing whitespace delimiters with null terminators (`'\0'`) & storing pointers in `argv[]` (ensuring `argv[0] == line`).

---

## test on host first

validate parser logic locally on laptop with unit tests:

```sh
# compile host test harness
gcc -Wall -Wextra -o cmdparse_host cmdparse_host.c cmdparse.c
# execute 60+ golden test assertions
./cmdparse_host
```

---

## 5 parsing stages & validation rules

1. `cmd_word_eq(const char *a, const char *b)`: case-insensitive string equality comparison. Walks both pointers in place without allocating or copying strings. Avoids `<ctype.h>` `toupper()`, which causes undefined behavior on signed `char` values $> \text{0x7F}$ on ARM architectures.
2. `cmd_parse_u8(const char *s, uint8_t *out)`: strict unsigned 8-bit integer parsing. Accepts numeric digit strings (`"0"` to `"255"`). Rejects negative numbers (`"-1"`), letters (`"12x"`), leading/trailing spaces (`" 7"`), empty strings, and numerical values exceeding 255 before integer wrapping.
3. `cmd_split(char *line, char *argv[], int max_args)`: in-place whitespace tokenizer. Collapses leading, trailing & multiple internal spaces. Overwrites delimiter spaces with `'\0'` and stores word pointers into `argv[]`. Returns total argument count, or `-1` if the number of words exceeds `max_args`.
4. `cmd_reader_feed(cmd_reader_t *r, char c)`: line accumulation state machine. Collects incoming UART bytes into `r->buf[]`. Strips carriage return (`'\r'`) before newline (`'\n'`). Accepts lines up to `CMD_MAX_LINE` (64 characters). If line length exceeds 64 characters, sets `dropping` flag, consumes all remaining bytes until `'\n'`, returns `-1`, and resets state cleanly so subsequent lines parse normally.
5. `cmd_parse(char *line, command_t *out)`: top-level command decoder. Tokenises line with `cmd_split()` and enforces argument count checks before dereferencing arguments. Validates commands (`PING` $\to 0$ args, `GET TEMP` $\to 1$ arg, `SET LED <index> <state>` $\to 3$ args), bounds-checks LED indices (`0` to `3`), and validates state keywords (`ON`, `OFF`). Returns specific status codes (`CMD_OK`, `CMD_ERR_EMPTY`, `CMD_ERR_UNKNOWN_VERB`, `CMD_ERR_ARG_COUNT`, `CMD_ERR_ARG_VALUE`, `CMD_ERR_ARG_RANGE`).

---

## optional: deploy & run on Pico W

once all host tests pass, you can optionally flash the demo firmware to a Pico W.
connect GP8 (TX1) & GP9 (RX1) to serial sender, with external LEDs wired to GP2, GP3, GP4 & GP5.

macOS:
```sh
# copy SDK import helper
cp ~/pico/pico-sdk/external/pico_sdk_import.cmake .
# configure & build
mkdir -p build && cd build
cmake -DPICO_BOARD=pico_w ..
make -j8 cmdparse
# flash to Pico W (bootsel mode)
cp cmdparse.uf2 /Volumes/RPI-RP2
# open USB CDC serial monitor (Ctrl-A Ctrl-\ to exit)
screen /dev/cu.usbmodem* 115200
```

windows (powershell):
```powershell
# copy SDK import helper
Copy-Item C:\pico\pico-sdk\external\pico_sdk_import.cmake .
# configure & build
New-Item -ItemType Directory -Force -Path build
cd build
cmake -DPICO_SDK_PATH="C:\pico\pico-sdk" ..
cmake --build . --target cmdparse
# flash to Pico W (bootsel mode)
Copy-Item cmdparse.uf2 -Destination D:\
```

---

## expected output

host test harness:
```
LAB 2 CHALLENGE - ASCII command parser

cmd_split
  "SET LED 3 ON" -> 4 words                      ok
    words are SET/LED/3/ON                       ok
  ragged spacing collapses to 4 words            ok
    words still SET/LED/3/ON                     ok
  "PING" -> 1 word                               ok
  "" -> 0 words                                  ok
  all spaces -> 0 words                          ok
  5 words with max 4 -> -1                       ok
  tokenised IN PLACE (argv points into line)     ok
cmd_parse_u8
  "0" -> 0                                       ok
  "7" -> 7                                       ok
  "007" -> 7                                     ok
  "255" -> 255                                   ok
  "256" rejected                                 ok
  "999" rejected                                 ok
  "" rejected                                    ok
  "12x" rejected                                 ok
  "x12" rejected                                 ok
  "-1" rejected                                  ok
  " 7" rejected                                  ok
cmd_word_eq
  "ON" == "ON"                                   ok
  "ON" == "on"                                   ok
  "Off" == "oFF"                                 ok
  "ON" != "OFF"                                  ok
  "ON" != "ONE"                                  ok
  "ONE" != "ON"                                  ok
  "" == ""                                       ok
cmd_parse - valid
  "PING"                                         ok
  "ping" (any case)                              ok
  "GET TEMP"                                     ok
  "SET LED 3 ON"                                 ok
  "SET LED 0 OFF"                                ok
  "set led 2 off"                                ok
  "  SET  LED  1  ON "                           ok
cmd_parse - rejected
  "" -> EMPTY                                    ok
  "    " -> EMPTY                                ok
  "SETLED 3 ON" -> VERB                          ok
  "BLINK" -> VERB                                ok
  "PING EXTRA" -> COUNT                          ok
  "GET" -> COUNT                                 ok
  "SET LED 3" -> COUNT                           ok
  "SET LED 3 ON X" -> COUNT                      ok
  "GET VOLTS" -> VALUE                           ok
  "SET PIN 3 ON" -> VALUE                        ok
  "SET LED x ON" -> VALUE                        ok
  "SET LED 3 MAYBE" -> VALUE                     ok
  "SET LED 4 ON" -> RANGE                        ok
  "SET LED 200 ON" -> RANGE                      ok
cmd_reader_feed
  "PING\r\n" -> line ready                       ok
    buffer holds "PING" (no CR)                  ok
  "PING\n" (bare LF) -> line ready               ok
    buffer holds "PING"                          ok
  no terminator yet -> 0                         ok
  empty line "\n" -> line ready                  ok
    buffer is empty string                       ok
  exactly CMD_MAX_LINE chars -> accepted         ok
    all characters kept                          ok
  one char too many -> -1                        ok
  recovers: next line parses                     ok
    buffer holds "PING"                          ok
  first of two lines                             ok
    holds "GET TEMP"                             ok
  second of two lines                            ok
    holds "SET LED 1 OFF"                        ok

ALL TESTS PASS  (0 failures)
```
