# efi-entry

UEFI boot entries from Linux without efibootmgr: list them, add one that boots another kernel image, boot an entry once, set an entry's load options (the kernel command line it passes), and delete one. It is one Python file that reads and writes the firmware's variables through efivarfs, so it needs nothing beyond Python, which Portage already brings. It replaces efibootmgr and its efivar library. On the laptop it is how every new kernel gets its one-time test boot, and how the SSD's host memory buffer is turned off at boot.

**Status:** in use on an HP OmniBook 7 with Gentoo (EFI stub kernel, no boot loader) since 2026-10-17: the one-time test boots of the kernel builds b23 to b29, and the load option on the default entry.

![Terminal output on a black background. "doas efi-entry show" prints BootCurrent: 0002, BootOrder: 0002,0001,9999, then the entries: Boot0001 Internal Hard Disk; Boot0002 Gentoo-min, booting \EFI\Gentoo\bzImage-min.efi with the load option nvme.max_host_mem_size_mb=0; Boot9999 USB Drive (UEFI). "efi-entry -h" then prints the usage: show, add NUM LABEL PATH [OPTIONS], next NUM, opts NUM [OPTIONS] and del NUM, with NUM in hexadecimal and EFI_TEMPLATE to copy another entry than BootCurrent.](docs/images/terminal.png)

## Features

- **`show`:** `BootCurrent`, `BootNext`, `BootOrder` and every `Boot####` entry with its label, the file it boots and its load options.
- **`add NUM LABEL PATH [OPTIONS]`:** a new entry that copies the entry you booted (same disk, same partition, same load options) and boots `PATH` on that partition instead, with `OPTIONS` appended to the load options. `EFI_TEMPLATE=NUM` copies another entry. The new entry isn't added to `BootOrder`.
- **`next NUM`:** boot that entry once. The firmware deletes `BootNext` when it uses it, so the boot after is the normal one again. A kernel that doesn't boot costs one power cycle.
- **`opts NUM [OPTIONS]`:** set an entry's load options, or remove them. For an EFI stub kernel they are appended to its built-in command line, so a kernel option can be changed without a rebuild. It prints the old and the new options.
- **`del NUM`:** delete an entry, and `BootNext` with it if it points there.
- **Checked writes:** every write is read back; `add` refuses an entry number that exists and a template without a file path or with binary load options (a Windows entry, for example) when there are options to append.

## How to install

Requires Linux with efivarfs mounted at `/sys/firmware/efi/efivars` (the default on a UEFI boot) and Python 3.

```sh
git clone https://github.com/r3clusionn/efi-entry
doas install -m 755 efi-entry/efi-entry /usr/local/sbin/
```

## How to use

A one-time test boot of a new kernel image next to the current one:

```sh
doas cp arch/x86/boot/bzImage /efi/EFI/Gentoo/bzImage-test.efi
doas efi-entry add F2 'Gentoo test (one boot)' '\EFI\Gentoo\bzImage-test.efi'
doas efi-entry next F2
doas reboot
# it works: copy it over the default image, then
doas efi-entry del F2
```

The SSD's host memory buffer off from the next boot on, without rebuilding the kernel (and back):

```sh
doas efi-entry opts 0002 nvme.max_host_mem_size_mb=0
doas efi-entry opts 0002
```

Paths use backslashes and are relative to the EFI system partition. Entry numbers are hexadecimal.

## How it works

- Each variable is a file named `<Name>-8be4df61-93ca-11d2-aa0d-00e098032b8c` (the EFI global variable GUID) in efivarfs: 4 bytes of attributes, then the data. New variables are written non-volatile, with boot and runtime access, the attributes boot entries have.
- efivarfs makes the files immutable so a stray write can't damage the firmware's settings. Before changing or deleting one, the tool clears the immutable flag with the `FS_IOC_SETFLAGS` ioctl, as efibootmgr's library does.
- A `Boot####` variable is an `EFI_LOAD_OPTION`: attributes, the length of the device path, a UCS-2 label, the device path, then optional data. `add` walks the template's device path nodes and replaces only the media file path node (type 4, subtype 4); the hard disk node, with the partition's number, offset and GUID, stays as it was, so the new entry boots from the same partition.
- For a Linux EFI stub the optional data is the command line as a NUL-terminated UCS-2 string. `opts` writes that string, and `show` prints the optional data only when it is one.

## Measurements

| | efi-entry | efibootmgr 18 |
|---|---|---|
| Packages | 0 (Python is already installed for Portage) | 2: efibootmgr and the efivar library (popt too where nothing else brings it) |
| Installed size | 5.3 KB | 741 KB (efibootmgr 101 KB, efivar 640 KB, without documentation) |

## Verification

- On the laptop, against the firmware: `add` with the entry booted (Boot0002) as template produced exactly the bytes of the earlier version of this tool, which made the one-time entries for the kernel test boots; `opts` set two options, then removed them; `next` set `BootNext` and `del` removed it together with the entry. `BootOrder`, `BootNext` and Boot0002 were byte-for-byte unchanged afterwards.
- `add` with `EFI_TEMPLATE=0001` (the firmware's "Internal Hard Disk" entry, which has no file path) was refused before anything was written.
- The one-time entries made with it booted: the kernel builds b23 to b29 were each tested once through `BootNext` before they replaced the default image.

## Limits

- No changes to `BootOrder` and no entries from scratch: a new entry is always a copy of an existing one with another file. For a first install, create the first entry with your distribution's tools or the firmware's setup screen.
- Only file paths on the template's partition; another disk or partition needs a template on it.
- Load options as text only (what an EFI stub and most boot loaders use).

## License

MIT (see `LICENSE`).
