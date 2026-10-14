# mt76 with an experimental MT7922 injected-rate fix

This is a fork of [openwrt/mt76](https://github.com/openwrt/mt76), the Linux MediaTek Wi-Fi driver. This fork's default `master` branch includes a focused MT7922 fix tested with Omdrop and an iPhone. It has not been accepted into upstream mt76 or released in a standard kernel.

## What the fix does

The PCI transmit path was losing the requested legacy transmission rate before constructing the packet's hardware descriptor. The fix saves that rate before TX status bookkeeping clears the overlapping flags, uses the driver's existing fixed-rate setup (including antenna/SPE settings), restores the original packet flags, and replaces only the descriptor's rate field.

The change is restricted to MT7922. It validates the band and rate index and excludes HT/VHT rates and hardware encapsulation. It preserves ACK policy. The local module-version marker and Omdrop channel-selection workaround are kept out of this driver patch.

- [Driver fix, commit `02b6c9a1`](https://github.com/shaecodes-wilde/mt76/commit/02b6c9a1ef87e55f2ccf90739e5dbcab55390cc7)
- [Changed PCI transmit function](mt7921/pci_mac.c)
- [Omdrop issue 14](https://github.com/t4t5/omdrop-owl/issues/14)
- [Test report, source hashes, and offline reproducer](https://github.com/shaecodes-wilde/omdrop-owl/tree/findings/mt7922-channel44/findings/mt7922-rate-upstream)

## What has been tested

The cleaned PCI function was compiled against matching Linux 7.2.5 sources and temporarily loaded on `7.2.5-3-omarchy`, retaining that kernel's common driver components. Hardware: MT7922 PCI `14c3:7922`, subsystem `1a3b:5916`; phone: iPhone 16 Pro Max. The exact iOS version remains unverified.

On channel 44, transfers completed in both directions in active-monitor and shared modes. The user confirmed received files opened on the iPhone; incoming files fully decoded on the computer. In shared mode, a 727,402-byte PNG upload took 3.69 seconds and a 2,801,190-byte incoming PNG took 4.16 seconds. These are individual transfer observations, not a performance benchmark.

Wi-Fi stayed associated in shared mode and passed 10/10 ping checks. Teardown restored the original driver, profile, and connection, with another 10/10 ping replies. Ethernet remained the default internet route. The cleaned driver was only loaded temporarily; merging this repository does not install it on the computer.

The 24-case offline regression suite was executed against three source bases: openwrt/mt76 `467ae2e3f190`, verified Linux 7.2.5, and the Linux maintainer mt76 tree at `2e6968e9286f`. Each unpatched baseline failed 11 cases; each patched version passed all 24. The suite exercises actual PCI/status/TXWI routines, stubbing only token allocation and DMA payload construction.

[t4t5 independently reported](https://github.com/t4t5/omdrop-owl/issues/14#issuecomment-6171706287) both transfer directions in both modes on MT7922 `14c3:0616` with the earlier experimental patch. That report does not establish independent testing of this cleaned revision.

## Build and test limits

The PCI components and offline modules build without warnings against the installed 7.2.5 headers. A complete current mt76 build needs newer mac80211 APIs, including `ieee80211_txq_aql_pending()`. A full current-kernel build, a reboot with this cleaned driver installed, longer reliability runs, and additional-chip support have not been validated.

Use driver sources and common components compatible with the target kernel. The offline reproducer instructions are in the linked test report. This repository does not provide an automatic installer or DKMS package. The driver patch also does not configure Omdrop's mode or radio channel; shared mode requires Wi-Fi on an AirDrop-compatible channel (6, 44, or 149).
