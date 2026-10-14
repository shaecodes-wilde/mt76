# mt76 with an experimental MT7920/MT7921/MT7922 injected-rate fix

This is a fork of [openwrt/mt76](https://github.com/openwrt/mt76), the Linux MediaTek Wi-Fi driver. This fork's default `master` branch includes an injected-rate fix for the MT7920/MT7921/MT7922 PCI path. Earlier revisions have transfer evidence on MT7921 and MT7922; the widened guard is build-checked and awaits confirmation on hardware. It has not been accepted into upstream mt76 or released in a standard kernel.

## What the fix does

The PCI transmit path was losing the requested legacy transmission rate before constructing the packet's hardware descriptor. The fix saves that rate before TX status bookkeeping clears the overlapping flags, uses the driver's existing fixed-rate setup (including antenna/SPE settings), restores the original packet flags, and replaces only the descriptor's rate field.

The change is restricted to MT7920, MT7921 (chip ID `0x7961`), and MT7922. MT7902 also uses this PCI driver but remains excluded. It validates the band and rate index and excludes HT/VHT rates and hardware encapsulation. It preserves ACK policy. The local module-version marker and Omdrop channel-selection workaround are kept out of this driver patch.

- [Original MT7922 driver fix, commit `02b6c9a1`](https://github.com/shaecodes-wilde/mt76/commit/02b6c9a1ef87e55f2ccf90739e5dbcab55390cc7)
- [Family guard revision, commit `2a2f085d`](https://github.com/shaecodes-wilde/mt76/commit/2a2f085dc1f920cabf8b8bc18686b9f974b27dac)
- [Changed PCI transmit function](mt7921/pci_mac.c)
- [Omdrop issue 14](https://github.com/t4t5/omdrop-owl/issues/14)
- [Test report, source hashes, and offline reproducer](https://github.com/shaecodes-wilde/omdrop-owl/tree/findings/mt7922-channel44/findings/mt7922-rate-upstream)

## Widened family guard, 2026-10-14

[verusferro independently tested the cleaned revision `02b6c9a1` with its MT7922 guard removed](https://github.com/t4t5/omdrop-owl/issues/14#issuecomment-6177921979) on MT7921 PCIe `14c3:7961`, driver `mt7921e`, CachyOS kernel 7.2.9. Only mt7921e was rebuilt; other mt76 components stayed stock. In active-monitor mode on channel 44, an iPad sent a 58,976,364-byte video to the PC in 37.0 seconds and received an 8,223,915-byte video in 20.3 seconds. Both transfers returned HTTP 200; the incoming video decoded cleanly.

The new revision enables the existing fix for MT7920/MT7921/MT7922 while retaining the explicit model guard. This source has no `is_mt7921()` helper. The production change is one condition; existing rate validation and descriptor handling are unchanged. MT7920 is included because it shares this preparation path, with no MT7920 hardware claim.

The PCI source and expanded 72-case offline regression module compile without warnings against `7.2.5-3-omarchy` headers. The expanded suite repeats the 23 descriptor cases for each intended chip and adds MT7902 exclusions on 2.4/5 GHz plus an unrelated-chip exclusion. A separate execution of the actual guard expression and source helpers checked every 16-bit chip ID and selected only the intended three. Driver and test diffs pass checkpatch with zero errors and warnings (`--no-tree --no-signoff`).

The expanded kernel test module has **not been executed**: this session cannot load modules without password-authenticated sudo. The exact widened revision has not been installed or tested with a phone. The existing 24-case pass results below belong to the earlier MT7922-only revision. [Build details and source hashes](https://github.com/shaecodes-wilde/omdrop-owl/blob/findings/mt7922-channel44/findings/mt7922-rate-upstream/family-validation.json) distinguish the new checks from those historical results.

## Earlier MT7922 validation

The cleaned PCI function was compiled against matching Linux 7.2.5 sources and temporarily loaded on `7.2.5-3-omarchy`, retaining that kernel's common driver components. Hardware: MT7922 PCI `14c3:7922`, subsystem `1a3b:5916`; phone: iPhone 16 Pro Max. The exact iOS version remains unverified.

On channel 44, transfers completed in both directions in active-monitor and shared modes. The user confirmed received files opened on the iPhone; incoming files fully decoded on the computer. In shared mode, a 727,402-byte PNG upload took 3.69 seconds and a 2,801,190-byte incoming PNG took 4.16 seconds. These are individual transfer observations, not a performance benchmark.

Wi-Fi stayed associated in shared mode and passed 10/10 ping checks. Teardown restored the original driver, profile, and connection, with another 10/10 ping replies. Ethernet remained the default internet route. That initial validation loaded the cleaned driver temporarily; merging this repository does not install it automatically.

The 24-case offline regression suite was executed against three source bases: openwrt/mt76 `467ae2e3f190`, verified Linux 7.2.5, and the Linux maintainer mt76 tree at `2e6968e9286f`. Each unpatched baseline failed 11 cases; each patched version passed all 24. The suite exercises actual PCI/status/TXWI routines, stubbing only token allocation and DMA payload construction.

[t4t5 independently reported](https://github.com/t4t5/omdrop-owl/issues/14#issuecomment-6171706287) both transfer directions in both modes on MT7922 `14c3:0616` with the earlier experimental patch. That report does not establish independent testing of this cleaned revision.

## Local adoption after validation

The same cleaned PCI function was subsequently installed on the test computer as DKMS `omdrop-mt7921e/7.2.5-airdrop2`, restricted to `7.2.5-3-omarchy`. Its installed and loaded srcversion both match `D5CFC98B9B40208307ABC3F`. Shared mode and a channel-44 Wi-Fi connection are configured persistently; Ethernet continues to provide internet. A subsequent 727,402-byte outgoing PNG uploaded in 2.63 seconds and opened on the iPhone. Another 63,738-byte JPEG fully decoded on the computer and the phone reported Sent. Wi-Fi passed another 10/10 ping checks. Reboot validation remains pending.

The incoming post-install check also exposed an Omdrop receiver completion issue: it sent HTTP 200 but kept the upload connection open until a 30-second timeout, leaving the phone on Sending. A separate local receiver change explicitly closes successful uploads while retaining delayed-chunk draining; its regression/protocol checks passed, and the next phone transfer reported Sent. That receiver change is separate from the mt76 patch and has not been published upstream.

## Build and test limits

The PCI components and offline modules build without warnings against the installed 7.2.5 headers. A complete current mt76 build needs newer mac80211 APIs, including `ieee80211_txq_aql_pending()`. A full current-kernel build, a reboot with this cleaned driver installed, longer reliability runs, and MT7920 hardware operation have not been validated. Exact-revision MT7921/MT7922 hardware confirmation and execution of the expanded suite remain pending.

Use driver sources and common components compatible with the target kernel. The offline reproducer instructions are in the linked test report. This repository does not provide an automatic installer or DKMS package. The driver patch also does not configure Omdrop's mode or radio channel; shared mode requires Wi-Fi on an AirDrop-compatible channel (6, 44, or 149).
