Prepping the IOS firmware upgrade for the WS-C2960G-24TC-L switch, closing out `Future-Planning/Open Items.md` #25.

## Context

The switch arrived with `c2960-lanbasek9-mz.122-50.SE5`, a 2010-era EOL IOS image with no modern SSH cipher support. Getting SSH working at all required pinning legacy KEX/cipher/MAC algorithms client-side (`Hardware Upgrade.md`, SSH troubleshooting steps), since the switch has nothing newer to offer. I'm upgrading the IOS image to close that gap instead of living with the client-side workaround indefinitely.

## Image research and decision

Two candidate images looked plausible by filename but turned out to be built for the wrong hardware:

- `c2960l-universalk9-mz.152-7.E14.bin`: built for the Catalyst 2960-L, not the 2960G. Different hardware architecture, won't boot on this switch.
- `c2960-lanbasek9-mz.152-7.E14.bin`: filename prefix matches, but the 15.2(x)E train targets 2960-Plus/2960-C hardware. The classic 2960G's PowerPC405/32MB flash can't run it; it would fail the kernel load and drop into ROMmon.

That left two valid EOL images for this exact hardware/license combination:

- `c2960-lanbasek9-mz.122-55.SE12.bin`: final maintenance build of the 12.2 branch, smaller image, leaves the most free DRAM, but no new SSH ciphers.
- `c2960-lanbasek9-mz.150-2.SE11.bin`: final release of the 15.0(2) branch, larger image, adds modern SSH/crypto algorithms.

I'm going with `150-2.SE11`. The point of this upgrade is getting rid of the legacy-crypto SSH workaround, not moving to a newer EOL image that still can't talk modern ciphers. Once this is flashed, Open Items #25 closes fully. Both the outdated-image half and the weak-crypto half, not just the first.

## Steps taken

1. Reviewed IOS image compatibility for the WS-C2960G-24TC-L against the currently running `12.2(50)SE5`, ruling out the two wrong-platform images above.
2. Downloaded and confirmed `c2960-lanbasek9-mz.150-2.SE11.bin` as the correct image for this hardware and license (LAN Base). Official Cisco MD5 for this file: `ca8eb99a9a084c7f07018c1fa3b33e21`. Not yet applied to the switch.

## Remaining steps

1. Check free flash space before transferring the new image. The 15.0(2)SE11 image runs larger (~12-14MB) than the 12.2 builds (~9.5-10MB), so this matters more here than it would for the other image:
   ```
   Switch# dir flash:
   ```
   Not deleting the active image until the new one is copied and verified.
2. Transfer `c2960-lanbasek9-mz.150-2.SE11.bin` to flash via TFTP/SCP/USB.
3. Verify image integrity against the known-good hash:
   ```
   Switch# verify /md5 flash:c2960-lanbasek9-mz.150-2.SE11.bin
   ```
   Expected: `ca8eb99a9a084c7f07018c1fa3b33e21`.
4. Point the boot variable at the new image and save:
   ```
   Switch# configure terminal
   Switch(config)# no boot system
   Switch(config)# boot system flash:c2960-lanbasek9-mz.150-2.SE11.bin
   Switch(config)# exit
   Switch# write memory
   Switch# show boot
   ```
5. Reload and confirm the switch boots on the new image (`show version`).
6. Regenerate SSH host keys now that 15.0 IOS is running, and confirm SSHv2:
   ```
   Switch# configure terminal
   Switch(config)# ip domain-name home.arpa
   Switch(config)# crypto key zeroize rsa
   Switch(config)# crypto key generate rsa modulus 2048
   Switch(config)# ip ssh version 2
   Switch(config)# end
   Switch# write memory
   ```
7. Re-test SSH from a normal client config, without the legacy KEX/cipher/MAC pins in `~/.ssh/config` (`Hardware Upgrade.md`). Those shouldn't be necessary anymore once modern ciphers are available.
