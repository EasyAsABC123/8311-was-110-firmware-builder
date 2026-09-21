# Safari 27 keep-alive proposal

Related issue: https://github.com/djGrrr/8311-was-110-firmware-builder/issues/54

This directory contains a proposed source patch for the uhttpd revision shipped
in `packages/basic/uhttpd_2020-10-01-3abcc891-1_mips_24kc.ipk`. It does **not**
replace that package, alter firmware assembly, or change a running modem.
Package integration is deliberately pending maintainer review in the draft PR.

## Behavior

uhttpd currently classifies any matching Safari UA as `UH_UA_SAFARI`, then
forces `r->connection_close = true` in `client_header_complete()`. The patch
adds a separate classification for Safari with a well-formed numeric product
`Version/27.0` or newer. That classification follows the normal connection
policy instead of the legacy forced-close case. Existing enum values and the
request structure layout are preserved.

- Preserve the legacy workaround for Safari below 27 and missing/malformed
  product versions. Do not infer product version from `Safari/604.1`.
- Retain existing Chrome/Opera/IE classification precedence. Known CriOS,
  FxiOS, EdgiOS and OPiOS tokens do not receive the new Safari exception.
- Never clear a prior close decision: explicit client close, HTTP/1.0 and
  globally disabled keep-alive still take precedence.
- Preserve the old IE POST workaround.

Safari 27.0 is the proposed minimum because that is the tested version. Newer
versions are allowed as a proposed compatibility policy; they have not been
measured. Desktop Safari 27 was not tested. A later CriOS experiment is documented below;
the source patch still excludes it pending a separate eligibility policy.
UA parsing is compatibility detection, not a security boundary.

## Evidence and limitations

A RAM-only uhttpd build at source revision
`3abcc89103799aaa79870fffcd58ec4370815024` used a process-local switch to skip
only Safari's forced-close assignment. The same authenticated iPhone dashboard,
HTTPS port, certificate, web root and network path were used for an off/on/off
comparison. Actual USB-reported iOS version was 27.0, build 24A437; Safari's UA
reported Version/27.0 even though its embedded CPU OS string said 18_7.

| Mode | Load event | First GPON status response |
| --- | ---: | ---: |
| Off 1 | 4.541 s | 9.857 s |
| Off 2 | 4.157 s | 9.863 s |
| On 1, fresh test process | 2.133 s | 2.984 s |
| On 2 | 0.415 s | 1.211 s |
| On 3 | 1.376 s | 2.286 s |
| Off again | 4.077 s | 9.242 s |

Median load time was 66.9% lower in the three on runs. This is a small
single-device sample, not a guarantee. Load events do not mean every dashboard
widget is ready; GPON response time is listed separately.

All off-mode responses used `Connection: close`; all on-mode responses used
`Keep-Alive`. A 45-second on capture completed 73 HTTP 200/304 responses and ten
GPON polls without JavaScript exceptions or failures tied to captured requests.
One unmatched cancellation during reload could not be attributed definitively;
similar events occurred while returning to stock behavior. Separate checks
passed static/login/ETag requests on one socket, explicit close, HTTP/1.0, old
IE POST close, and 20-second idle timeout/reconnection. Date-based
If-Modified-Since validation remains a separate unverified path.

The hardware A/B used the unconditional **test switch**, not this version
parser. This proposal is separately checked by native C parser/policy tests and
a complete cross-compilation of uhttpd with the MIPS32r2 soft-float SDK. The
version-gated executable has not been deployed or tested on hardware. A full
firmware build, flash, older-browser matrix and extended soak remain pending.

All temporary test changes were removed, including the management test-port
rule. The filter rule listing matched its original state and production uhttpd
retained its original process and binary. No private hostnames, credentials,
session cookies or packet payloads are included here.

## Validate the source patch

Use pristine uhttpd source at the shipped revision:

```sh
git clone https://github.com/openwrt/uhttpd.git /tmp/uhttpd-safari-review
git -C /tmp/uhttpd-safari-review checkout 3abcc89103799aaa79870fffcd58ec4370815024
python3 tools/uhttpd/test-safari.py /tmp/uhttpd-safari-review
```

The test checks the original source hashes, applies the patch in a temporary
copy, and compiles the actual patched UA parser and close-policy switch with
minimal transport stubs. It exercises current/older/future/malformed versions,
known alternative browser tokens, pre-existing close decisions, explicit close,
POST behavior and legacy IE. It does not emulate the full HTTP transport; the
separate device experiment supplied that evidence for the switch-based build.
Python 3, `patch` and a host C compiler are required; tests do not contact a
modem or change the input source tree.

## Proposed package integration (not performed by this PR)

The firmware builder consumes prebuilt IPKs rather than compiling uhttpd in
`build.sh`. After agreeing on the version policy, maintainers can place
`patches/100-safari-27-keepalive.patch` in their OpenWrt uhttpd recipe's patch
directory (normally `package/network/services/uhttpd/patches/`), keeping the
source revision above and increasing the package release from 1 to 2.

Rebuild uhttpd and its Lua/ubus packages with the firmware's compatible MIPS
configuration, including its no-MIPS16 requirement. Validate the resulting
package set with an isolated device instance before replacing the three
matching IPKs in `packages/basic/`. Do not overwrite modem libraries with SDK
link-time copies. Once reviewed and tested, the rebuilt packages make the
change part of a normal firmware image. Reverting this patch and rebuilding
restores the original Safari workaround.

## CriOS follow-up

Chrome 153.0.8010.24 on iOS 27.0 showed the same benefit in a separate completed
switch-based off/on/off test: stock dashboard loads 6.719 / 4.638 / 4.472 s;
keep-alive loads 2.278 / 0.403 / 0.388 s. A 45-second keep-alive capture completed
73 HTTP 200/304 responses without network errors or JavaScript exceptions.
The Chrome test was conducted through one continuous USB inspector connection;
failed reconnect attempts were excluded. All temporary changes were removed.

The patch still excludes CriOS because Chrome's application version is not a
WebKit version. See the combined upstream report and version-policy discussion:
https://github.com/openwrt/uhttpd/issues/42
