# DirtyFrag APatch integration

This source tree adds APatch as an alternative manager while retaining the existing KernelSU path.

## Runtime flow

1. The manager picker recognizes an installed APK with `libapd.so` in `ApplicationInfo.nativeLibraryDir`.
2. Before DirtyFrag starts its existing exploit flow, `ExploitRunner` reads the running kernel release and derives the KMI using the same pattern as APatch (for example `android14-5.15`).
3. It opens the selected manager APK's `assets/<KMI>_kernelpatch.ko` through a package context and extracts it to DirtyFrag's device-protected app directory as `kernelpatch.ko`.
4. It stores `manager_type=apatch`, the absolute `libapd.so` path, and the extracted `.ko` path in device-protected `dfroot` preferences.
5. After the existing DirtyFrag Bootstrap has started in its privileged context, it exports `APATCH_APD`, `APATCH_KO`, and `APATCH_PACKAGE` and starts a child `/system/bin/sh`.
6. The child executes:

   ```sh
   "$APATCH_APD" late-load --module "$APATCH_KO" --package-name "$APATCH_PACKAGE"
   ```

7. On success, the existing `/dev/dfm6` success marker is written. If soft reboot is enabled, Bootstrap calls `libapd.so soft-reboot`.

## Important path detail

`base.apk/assets/<name>.ko` is a ZIP entry, not a normal filesystem file. APatch's `late-load --module` implementation reads the supplied module path with `fs::read()`. Passing a string such as `/data/app/.../base.apk/assets/android14-5.15_kernelpatch.ko` therefore cannot work as a regular file path. The integration extracts the matching asset to a real file first, then passes that file path to `--module`.

The APatch source supplied with this change defines `late-load` options as `--module`, `--kmi`, and `--package-name`. This integration uses `--module` and `--package-name`; the module itself is selected by the APK asset's KMI name.

## Compatibility notes

- The selected APatch APK must include `libapd.so` and the asset matching the device KMI.
- The APatch source's current asset list covers `android12-5.10`, `android13-5.10`, `android13-5.15`, `android14-5.15`, `android14-6.1`, `android15-6.6`, and `android16-6.12`. A device outside the APK's asset list will fail during asset extraction.
- APatch's `late-load` path loads KernelPatch with its own relocation logic and applies its live SELinux policy. It is not equivalent to a normal `insmod`.
- This source change has not been built into an APK in this environment: the Gradle wrapper could not download Gradle because network access was unavailable. The modified Bootstrap C source passes host `gcc -fsyntax-only`; an Android build and device validation are still required.
