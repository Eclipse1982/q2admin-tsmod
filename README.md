# Q2Admin TSmod (Eclipse Fork)

Forked from [tastyspleen/q2admin-tsmod](https://github.com/tastyspleen/q2admin-tsmod).

Q2Admin is a transparent proxy mod for Quake 2 that adds admin functions and anti-cheat detection. It sits between the Q2 server engine and the game mod (e.g., TMG, CTF, vanilla DM), intercepting communication to add its features without modifying the underlying mod.

**Base version:** Q2Admin 1.17.48-tsmod-2 (TastySpleen's fork of Q2Admin 1.17.44 with R1CH's security patches and false-positive bot detection fixes for laggy players).

## Changes in This Fork

### Bug Fixes (2026-03-28)

**Critical: Out-of-bounds array writes in log file validation (zb_log.c)**
- The log file number validation used `||` instead of `&&` in 5 separate locations (lines 202, 275, 295, 941, 966), making the condition always evaluate to true. Any integer would pass validation, allowing writes outside the `logFiles[32]` array. This could cause crashes or memory corruption from a malformed `q2adminlog.txt` or via the `!logevent` admin command.

**Critical: Command queue buffer overflow (zb_msgqueue.c)**
- `addCmdQueue()` wrote to the command queue array *before* checking if the index was within bounds. If `maxCmds` reached `ALLOWED_MAXCMDS` (50), the function would write to index 50 (out of bounds) before the safety check at 45 could trigger.
- The function could also be called with `client = -1` from the vote system (`zb_vote.c`), causing a write to `proxyinfo[-1]` (undefined behavior). Added a guard for invalid client indices.

**Critical: Broken checkvar file parsing (zb_checkvar.c)**
- `ReadCheckVarFile()` had premature `continue` statements after setting the checkvar type (`CV_CONSTANT` or `CV_RANGE`). This caused the parser to skip the variable name, value, and range parsing entirely, and never increment `maxcheckvars`. The entire checkvar feature (enforcing client-side cvar values) was non-functional.
- Additionally, the successful parse paths for both constant and range types used `continue` which skipped the `maxcheckvars++` increment at the bottom of the loop. Fixed by adding the increment before each `continue`.

**Memory corruption in realloc (zb_util.c)**
- `q2admin_realloc()` copied `newsize - oldsize` bytes instead of `oldsize` bytes when growing an allocation. This copied too few bytes, losing data from the end of the original allocation.

**Broken last-key matching in Info_ValueForKey (zb_util.c)**
- Dead code `if (!*s) return ""` inside the value-parsing loop would prevent the function from ever matching the last key-value pair in a userinfo string (where there is no trailing backslash). Removed the dead branch so the parsed value is correctly checked against the key.

**Buffer overflow in admin command logging (zb_zbot.c)**
- `ADMIN_process_command()` used `sprintf` to write `gi.argv(0)` + `gi.args()` (unbounded server input) into a 256-byte buffer. Replaced with `snprintf`.

**Buffer overflow in !dostuff command (zb_zbot.c)**
- The `!dostuff` admin command used `strcpy`/`strcat` with no bounds checking to concatenate an arbitrary number of arguments into a 512-byte buffer. Replaced with `snprintf` with bounds tracking in both the "all clients" and single-client code paths.

**Buffer overflow in stuff_private_commands (zb_zbot.c)**
- `sprintf(temp, "%s\r\n", ...)` could write up to 258 bytes into a 256-byte buffer (the command field is 256 bytes + 2 bytes for `\r\n`). This was flagged as a compiler warning. Replaced with `snprintf`.

**Write to read-only file handle (zb_log.c)**
- `displayLogFileCont()` opened a log file with `fopen(logname, "rt")` (read mode), read from it, then called `fprintf()` to write to the same handle. Removed the erroneous write.

## Building

```bash
# Linux x86_64
make all

# The output is gamex86_64.so
```

## Installation

Q2Admin acts as a proxy between the Q2 engine and your game mod:

1. Rename your existing game library: `gamex86_64.so` -> `gamex86_64.real.so`
2. Copy Q2Admin's `gamex86_64.so` into the mod directory
3. Copy the `q2admin*.txt` config files into the mod directory
4. Configure `q2admin.txt` (set `serverip`, admin password, etc.)
5. Restart the server

Q2Admin will load as the game library, then chain-load your real mod from `gamex86_64.real.so`.

## License

GPL v2 (inherited from original Q2Admin by Shane Powell)
