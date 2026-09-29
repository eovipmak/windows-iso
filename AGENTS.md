# AGENTS.md

This repo is Windows unattended-install assets, not application code. No README, no build/test/lint/CI, no task runner. The `.iso` files are local-only (matched by `.gitignore`); everything else is tracked.

## Layout

- `autounattend.xml` (root) — built answer file for the Windows 10 LTSC ISO. Byte-identical to `windows-10-ltsc/autounattend.xml`; keep both copies in sync. For the Server 2019 ISO, copy `windows-server-2019/autounattend.xml` to the ISO root instead.
- `windows-10-ltsc/` — editable sources for Win10 Enterprise LTSC (`/Index:1`): `autounattend.xml` + 15 payload files (`.ps1`, 2 scheduled-task `.xml`, 2 `cloudbase-init*.conf`).
- `windows-server-2019/` — editable sources for Windows Server 2019 Standard Evaluation (Desktop Experience, `/Index:2`): the same 15 payload files plus `autounattend.xml` and `ADDED_COMPONENTS.txt` (media-root build doc, not an embedded payload), with Server-specific tweaks (disable Server Manager at logon, Shutdown Event Tracker, IE Enhanced Security) in `Specialize.ps1`.
- `windows_10_enterprise_ltsc.iso` (~4.7G), `windows_server_2019.iso` (~5.7G) — local-only, matched by `.gitignore` (`*.iso`). Never `cat`/`diff`/`read` them, never force-add them.
- Both `.iso` files are **built unattended media**, not stock images: each has `autounattend.xml`, `$WinPEDriver$` (virtio-win 0.1.302), `CloudbaseInitSetup_x64.msi` (Cloudbase-Init 1.1.8) and `ADDED_COMPONENTS.txt` at the ISO root. `windows_server_2019.iso` was previously named `windows_server_2019_unattended.iso`; rebuilds are done in place under this name (recipe below). The Server 2019 full unattended install was validated in QEMU/OVMF.

## Source-of-truth rule (verified)

Payload files in each OS dir are embedded in that dir's `autounattend.xml` under `<Extensions><File path="C:\Windows\Setup\Scripts\...">`; all payloads must be embedded as **escaped text** so the generator's `ExtractScript` (`$file.InnerText.Trim()`) reconstructs them byte-for-byte. Never embed `.xml` payloads as live XML subtrees: `InnerText` drops markup.

- Edit the file in the OS dir, then re-embed into that dir's `autounattend.xml` (and, for Win10, the root copy too). Never edit only one side.
- `ADDED_COMPONENTS.txt` is a plain media-root document (grafted at ISO build time), not an `ExtractScript` payload; it is not embedded in any answer file.
- `windows-server-2019/` embeds all 15 payloads as text and is verified in sync. The Win10 pair still embeds `ShowAllTrayIcons.xml` / `MoveActiveHours.xml` as live subtrees (known defect: those two files materialize corrupted; task registration fails harmlessly) — fix by converting them to escaped text when touching that template.
- Regenerate via https://schneegans.de/windows/unattend-generator/ using the full parameter string in the `<!-- ... -->` comment at the top of `autounattend.xml` (line 3), then re-apply the Server tweaks. Hand-editing the embedded copy risks breaking the encoding contract (UTF-8 for `.ps1`/`.xml`, UTF-16 for `.reg`/`.vbs`/`.js` per `ExtractScript`).
- The `.ps1` files are intentionally single-line/minified — do not reformat.
- Re-embedding a payload that contains `&`, `<` or `>` must XML-escape it (`&amp;`, `&lt;`, `&gt;`) — the `.ps1` payloads are full of `&` (call operator) and `*>&1`. Writing the file's raw text into the `<File>` element breaks the document (`not well-formed (invalid token)`); the untouched payloads in `autounattend.xml` show the `&amp;`/`&gt;` convention. The sync check reads via `itertext()`, which unescapes, so escaped text compares equal to the standalone file.

## ISO build (verified)

The install media is the stock Microsoft ISO re-mastered with additions at the root: `autounattend.xml`, `$WinPEDriver$`, `CloudbaseInitSetup_x64.msi`, `ADDED_COMPONENTS.txt`. Because `windows_server_2019.iso` is itself the built media (there is no separate `_unattended.iso`), rebuild it from the previous built media to a temp file, then swap it in:

```bash
# Previous built media = base; Win10 media = source of the added components.
mount -o loop,ro windows_server_2019.iso /mnt/ms
mount -o loop,ro windows_10_enterprise_ltsc.iso /mnt/ms10

genisoimage -o windows_server_2019.iso.new \
  -V 'SSS_X64FREE_EN-US_DV9' -iso-level 3 -udf -allow-limited-size \
  -b boot/etfsboot.com -no-emul-boot -boot-load-size 8 \
  -eltorito-alt-boot -e efi/microsoft/boot/efisys.bin -no-emul-boot -boot-load-size 1 \
  -m autounattend.xml \
  -graft-points /=/mnt/ms/ /autounattend.xml=windows-server-2019/autounattend.xml \
    '/$WinPEDriver$=/mnt/ms10/$WinPEDriver$' \
    /CloudbaseInitSetup_x64.msi=/mnt/ms10/CloudbaseInitSetup_x64.msi \
    /ADDED_COMPONENTS.txt=windows-server-2019/ADDED_COMPONENTS.txt

umount /mnt/ms /mnt/ms10
mv windows_server_2019.iso.new windows_server_2019.iso    # never overwrite the media while mounted
```

- The extra root files are not optional: the PE script DISM-injects `$WinPEDriver$` into the applied image, `Specialize.ps1` re-runs it via `InstallVirtioDrivers.ps1`, and `FirstLogon.ps1` installs `CloudbaseInitSetup_x64.msi` via `InstallCloudbaseInit.ps1`. Without them both steps log "Skipping"/"ERROR" and the image is incomplete. Source them from the Win10 media (as above) or upstream (virtio-win ISO, cloudbase.it).
- `-m autounattend.xml` keeps the base media's stale answer file from clashing with the explicit graft.
- `-udf -allow-limited-size` is required: the 4.7 GB `install.wim` exceeds ISO9660's 32-bit file size and UDF carries the true size (verified via kernel mount and DISM apply). The UDF namespace also preserves `$WinPEDriver$` verbatim; the media has no Joliet/Rock Ridge, and Windows mounts it as UDF.
- Boot entries mirror the original catalog: BIOS `boot/etfsboot.com` (8 × 512-byte sectors), UEFI `efi/microsoft/boot/efisys.bin` (platform 0xEF, 1 sector).
- Do not use xorriso for this media: it parses only the ISO9660 stub (a lone `README.TXT`) and would drop the UDF tree.
- **Rebuilding from the already-built media** (the normal case, since the base is itself the built ISO): the base already carries `$WinPEDriver$`, `CloudbaseInitSetup_x64.msi` and `ADDED_COMPONENTS.txt`, so re-grafting them from the Win10 media fails with `genisoimage: Error: ... have the same Joliet name`. genisoimage still builds a Joliet tree even though the output is UDF, and `-joliet-long` does not help. Graft only the changed files and exclude the base's stale copies; `-m` drops the directory-tree copy while the explicit graft still wins (this is why the `-m autounattend.xml` line works):

  ```bash
  mount -o loop,ro windows_server_2019.iso /mnt/ms
  genisoimage -o windows_server_2019.iso.new \
    -V 'SSS_X64FREE_EN-US_DV9' -iso-level 3 -udf -allow-limited-size \
    -b boot/etfsboot.com -no-emul-boot -boot-load-size 8 \
    -eltorito-alt-boot -e efi/microsoft/boot/efisys.bin -no-emul-boot -boot-load-size 1 \
    -m autounattend.xml -m ADDED_COMPONENTS.txt \
    -graft-points /=/mnt/ms/ /autounattend.xml=windows-server-2019/autounattend.xml \
      /ADDED_COMPONENTS.txt=windows-server-2019/ADDED_COMPONENTS.txt
  umount /mnt/ms && mv windows_server_2019.iso.new windows_server_2019.iso
  ```

## Testing in QEMU/OVMF (verified)

How the Server 2019 install was validated end-to-end under KVM (~20 min; WinPE has no serial console, so watch the screen):

```bash
cp /usr/share/OVMF/OVMF_VARS_4M.fd vars.fd          # OVMF_CODE_4M.fd must be read-only
qemu-img create -f qcow2 disk.qcow2 40G
qemu-system-x86_64 -machine pc,accel=kvm -cpu host -m 4096 -smp 2 \
  -drive if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF_CODE_4M.fd \
  -drive if=pflash,format=raw,file=vars.fd \
  -drive file=windows_server_2019.iso,media=cdrom,readonly=on,if=ide,index=2 \
  -drive file=disk.qcow2,if=ide,index=0 -boot order=d,menu=off \
  -display none -vga std -daemonize -qmp unix:qmp.sock,server,nowait
```

- An empty 40 GB IDE disk satisfies the PE script's 32–60 GB single-disk check; the install then runs unattended to the desktop (FirstLogon included).
- The media prompts "Press any key to boot from CD or DVD" (`efisys.bin`): send a key (`spc`) repeatedly for the first ~10 s, then `ret` at the Windows Boot Manager menu. Without a key OVMF falls through to PXE.
- Progress checks: QMP `screendump` (PPM) — winPE/setup screenshots; a PPM→PNG converter in pure Python avoids needing ImageMagick. Key events via QMP `send-key`; chords such as Ctrl+Alt+Del must be **one** call with all keys (`ctrl`,`alt`,`delete`) — separate calls do not form a chord.
- Inspect the installed disk offline: `qemu-nbd --connect=/dev/nbd0 disk.qcow2`, `partx -a /dev/nbd0`, `ntfs-3g -o ro /dev/nbd0p3 /mnt` (Windows = partition 3; the qcow2 must be disconnected from any earlier qemu-nbd/VM first). `Windows/Setup/Scripts/*.log` are UTF-16LE (`iconv -f UTF-16LE`); `Windows/System32/Tasks/<name>` proves a task registered.
- Proof the media additions were consumed (verified): `InstallVirtioDrivers.log` has `Found driver folder at D:\$WinPEDriver$` + `pnputil exit code: 0` (43 packages), and `InstallCloudbaseInit.log` has `Installing D:\CloudbaseInitSetup_x64.msi` + `msiexec exit code: 0`, with `Program Files\Cloudbase Solutions\Cloudbase-Init\` present. `FirstLogon.log` registering `MoveActiveHours` and deleting `C:\Windows\Panther\unattend.xml` confirms the rest of the pass.
- The installed Administrator password is `Vinacafe@2026` (see password gotcha below).

## On-VM execution flow (setup passes)

1. `windowsPE`: generated `pe.cmd` wipes/partitions the target disk, applies the image index from `InstallFromIndex` (`/Index:1` Win10 LTSC, `/Index:2` Server 2019 Desktop Experience), injects `$WinPEDriver$`/VirtIO drivers, copies answer file to `W:\Windows\Panther\unattend.xml`.
2. `specialize`: `ExtractScript` materializes payloads → `Specialize.ps1` (debloat, hardening, disables password complexity via `secedit`) → `DefaultUser.ps1` runs against the `HKU\DefaultUser` hive (load/edit/unload).
3. `FirstLogon.ps1`: installs VirtIO guest tools → installs/configures Cloudbase-Init → deletes `unattend.xml`, `unattend-original.xml`, `C:\Windows.old`. Logs to `C:\Windows\Setup\Scripts\*.log`.

## Gotchas

- **Destructive by design**: PE script runs `diskpart CLEAN` on the single empty disk sized 32–60 GB (`TargetDiskMinSize/MaxSize`), GPT/UEFI only. Do not loosen disk criteria or change `PartitionLayout` without understanding the safety check.
- **Secrets in repo**: Administrator password is in `autounattend.xml` twice (base64-obscured `AdministratorPassword`/`AutoLogon`) and in cleartext in the generator-URL comment (`BuiltinAdministratorPassword=...`). Treat as compromised; rotate rather than reuse for new images.
- **Generator password encoding**: with `ObscurePasswords=true` the generator stores `base64(UTF16LE(password + elementName))`, e.g. decoding the `AdministratorPassword` value yields `Vinacafe@2026AdministratorPassword`. Windows applies the plain password (`Vinacafe@2026`). A decoded element-name suffix is expected — do not "fix" it.
- **Cloudbase-Init scope**: `windows-server-2019/` uses the `CloudStack` datasource only (`[cloudstack] metadata_base_url=http://192.168.1.1/`; environment-specific — `10.1.1.1` is the cloudbase-init default), `SetUserPasswordPlugin` enabled, `CreateUserPlugin` removed. `cloudbase-init-unattend.conf` intentionally has a reduced plugin set (MTU + hostname). Keep both `.conf` files paired with `InstallCloudbaseInit.ps1` expectations (MSI names `CloudbaseInitSetup_x64.msi` / `CloudbaseInitSetup_1_1_8_x64.msi` scanned on drives D–Z).
- **Cloudbase-Init password injection (verified)**: a simple injected password disappears — Administrator silently ends up with a random password. Two traps: (1) `CreateUserPlugin` runs first and resets the existing `Administrator` to a random 20-char password (`rename_admin_user` defaults false); (2) when the active datasource is `ConfigDriveService`, `can_update_password` is false and `get_admin_password()` finds no `admin_pass` (CloudStack config drive carries only `vm_password.txt`/`vendor_data.json`), so `SetUserPasswordPlugin` generates another random password. The CloudStack password server lives on the VR (port 8080) and is only used via the `CloudStack` datasource. Third trap: Windows password-complexity makes `NetUserSetInfo` reject the simple password, so `Specialize.ps1` sets `PasswordComplexity = 0` via `secedit`. Diagnose in `C:\Program Files\Cloudbase Solutions\Cloudbase-Init\log\cloudbase-init.log`: `Metadata service loaded: '<name>'`, `Generating a random user password`, `Updating password is not required.`, `Password succesfully updated`. For CloudStack the plugin re-runs each boot, so a reset queued in CloudStack applies on the next boot; a randomized password is unrecoverable via ConfigDrive (offline SAM reset needed).
- **VirtIO**: the media ships `$WinPEDriver$` (virtio-win 0.1.302); the PE script `drvload`s it and runs DISM `/Add-Driver /Recurse` against the applied image, and `Specialize.ps1` re-runs it via `pnputil` (`InstallVirtioDrivers.ps1`). `VirtIoGuestTools.ps1` separately looks for `virtio-win-guest-tools.exe` at the root of any attached drive at FirstLogon (it is not on the ISO). Server 2019 is build 17763, so the `w10`/`2k19` driver folders are the correct ones. If the media lacks `$WinPEDriver$` or the MSI, those steps log "Skipping"/"ERROR" — keep the root additions when rebuilding.
- **Server 2019 ISO is Evaluation media** (`sources/EI.CFG` → `[Channel] eval`): installs expire after 180 days and cannot be activated with retail/volume keys. `install.wim` indexes: 1 = Standard Core, 2 = Standard Desktop Experience, 3 = Datacenter Core, 4 = Datacenter Desktop Experience; the answer file pins `/Index:2`. There are no `.clg` files on this repack; Setup still works.
- **MoveActiveHours registration**: in `windows-server-2019/`, `FirstLogon.ps1` registers the task, not `Specialize.ps1` — `Register-ScheduledTask` fails with `HRESULT 0x8004100a` during the specialize pass (Task Scheduler provider not ready yet), but succeeds at first logon (verified). Do not move it back.
- **Task XML BOM**: `ExtractScript` writes `.xml` payloads as UTF-8 **with BOM**. `Register-ScheduledTask -Xml (Get-Content -Raw)` accepts that, but `schtasks.exe /Create /XML <file>` fails with "The task XML is malformed. (1,2)" — use the PowerShell cmdlet.
- **install.wim repack quirk**: on this Server 2019 ISO the WIM header's XML offset (`rhXmlData`) is bogus (points at compressed data); the real UTF-16 `<WIM>` XML with the image indexes sits near the end of the file. Kernel UDF mount, 7z and DISM all read the WIM fine — only direct header parsing is affected.

## Verification (no test suite)

```bash
python3 -c "import xml.etree.ElementTree as ET; [ET.parse(p) for p in ('autounattend.xml','windows-10-ltsc/autounattend.xml','windows-server-2019/autounattend.xml')]; print('XML OK')"
cmp autounattend.xml windows-10-ltsc/autounattend.xml && echo "Win10 copies in sync"

# Embedded payloads must equal the standalone files (ExtractScript uses InnerText.Trim()).
python3 - <<'EOF'
import xml.etree.ElementTree as ET
NS = '{https://schneegans.de/windows/unattend-generator/}'
for d in ('windows-10-ltsc', 'windows-server-2019'):
    t = ET.parse(f'{d}/autounattend.xml')
    bad = [f.get('path').rsplit('\\', 1)[-1] for f in t.getroot().findall(f'.//{NS}File')
           if ''.join(f.itertext()).strip() != open(f"{d}/{f.get('path').rsplit(chr(92), 1)[-1]}", encoding='utf-8-sig').read().strip()]
    print(d, 'payloads OK' if not bad else f'MISMATCH: {bad}')
EOF
```

Built media content check (root must carry `autounattend.xml`, `$WinPEDriver$`, `CloudbaseInitSetup_x64.msi`, `ADDED_COMPONENTS.txt`; mount as UDF, the only namespace Windows uses):

```bash
mount -t udf -o loop,ro windows_server_2019.iso /mnt/ms && ls /mnt/ms && umount /mnt/ms
mount -t udf -o loop,ro windows_10_enterprise_ltsc.iso /mnt/ms && ls /mnt/ms && umount /mnt/ms
```

Full install can only be validated in a VM (UEFI, empty 32–60 GB disk, `install.wim`/`esd`/`swm` with the requested index present); follow the QEMU/OVMF recipe above. There is no local check for that.
