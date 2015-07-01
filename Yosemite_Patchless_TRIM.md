# Yosemite Patchless TRIM Support

Apple started laying the groundwork for official 3rd party TRIM support as early as OS X 10.10.3, when they changed their IOAHCIBlockStorage.kext to understand a new Info.plist option that allows the user to force their driver to enable TRIM on non-Apple SSDs.

Officially, the hidden option is "supposed" to be enabled via their new "trimforce" tool, but all their tool does is copy a new file called "AppleDataSetManagement.kext" into the system folder. That kext in turn just injects the new option into IOAHCIBlockStorage.kext at runtime.

The advantages of this new injection method over the classic binary patching is that this is the clean, official Apple method and is guaranteed to work as long as you are on 10.10.3 or higher (lower versions do _not_ work).

On June 30th, 2015, Apple officially released OS X 10.10.4 with the "trimforce" command built-in. And OS X 10.11 El Capitan users have the command as well (simply disable Rootless, run "sudo trimforce enable", and re-enable Rootless). _Hopefully Apple improves trimforce before El Cap's release, so that it no longer requires us to temporarily disable rootless. That seems like an oversight on their part. Unfortunately it's still not fixed as of Developer Preview 2 from late June 2015._

**Disclaimer**: Enabling TRIM is done at your own peril and may cause data corruption if you're using an old SSD with a buggy TRIM command. You cannot make any claims against Apple or myself if something bad happens with your SSD.


## Enabling TRIM Support on OS X 10.11 El Capitan Beta

* Since El Capitan includes the "Rootless" system security feature, we temporarily need to disable it before we can enable TRIM via Apple's tool.
* Open a Terminal window and run the following command to temporarily _disable Rootless_:
```
sudo nvram boot-args="rootless=0"
```
* _Restart_ your machine so that Rootless is turned off, and then run these commands so that Rootless will be enabled again on the next reboot, and enable TRIM:
```
sudo nvram -d boot-args
sudo trimforce enable
```
* The machine will automatically _restart_. You now have official TRIM support, and Rootless is enabled again as well.


## Enabling TRIM Support on OS X 10.10.4+ Yosemite

* Open a Terminal window and run the following command to _enable_ TRIM:
```
sudo trimforce enable
```
* The machine will automatically _restart_. You now have official TRIM support. This is the Apple method, so you _no longer_ have to worry about nvram (kext-dev-mode), binary or Info.plist patches, software updates removing TRIM support, booting to a gray stop sign error, and so on. Everything is properly codesigned by Apple and this permanently enables TRIM via the official method!
* If you ever want to disable TRIM again, just run the trimforce command with the "disable" option instead.


## Enabling TRIM Support on OS X 10.10.3 Yosemite (same method as "trimforce")

* The "trimforce" tool doesn't exist on 10.10.3, but the kext that the tool installs can still be manually installed.
* Download the official [AppleDataSetManagement.kext](http://www72.zippyshare.com/v/BQFjtD3i/file.html) (properly codesigned by Apple). Use the big, red "Download Now" button on the top right and save it to your Downloads folder.
* (_Optional_) Verify that your downloaded ZIP file is intact:
```
ADSM_SHA1=$(openssl sha1 ~/Downloads/AppleDataSetManagement.zip | cut -d'=' -f2 | cut -d' ' -f2); [ "${ADSM_SHA1}" = "1df56eeef3499e22eb5072dc481bce8a3d2413a7" ] && echo -e "\n* AppleDataSetManagement.zip is valid. It is safe to proceed with the installation now." || echo -e "\n* AppleDataSetManagement.zip is INVALID. Do not install it."
```
* Install it to /System/Library/Extensions:
```
sudo unzip ~/Downloads/AppleDataSetManagement.zip -d /System/Library/Extensions
sudo touch /System/Library/Extensions
```
* Restart your machine. You will now have official TRIM support with the exact same stability as if you had used the "trimforce" command. No need to worry about system updates and so on.
* If you ever want to _disable_ TRIM again, just run these two commands and reboot the machine:
```
sudo rm -rf /System/Library/Extensions/AppleDataSetManagement.kext
sudo touch /System/Library/Extensions
```


## Enabling TRIM Support on OS X 10.10.3 Yosemite (Completely Manual Method)

* This manual technique is just included here for those who are interested in seeing steps for the exact injection work that the "trimforce" kext does. Its drawbacks are that it modifies the kext and therefore breaks the kext signature (so you need to run with kext-dev-mode), and that the setting might get lost after system updates. Most people should use the methods above instead!
* First, remove any existing binary patch (if you've patched it with TRIM Enabler, Disk Sensei or Chameleon, for instance).
* Next, make sure that _kext signature checks are disabled_ by running the command below:
```
sudo nvram boot-args="kext-dev-mode=1"
```
* _Restart_ your machine so that kext-dev-mode takes effect.
* _Verify_ that kext-dev-mode is active by typing this command and looking for "kext-dev-mode" in the output:
```
sudo nvram boot-args
```
* If you want the ability to disable TRIM later, you should now make a _backup_ before proceeding:
```
sudo cp /System/Library/Extensions/IOAHCIFamily.kext/Contents/PlugIns/IOAHCIBlockStorage.kext/Contents/Info.plist /System/Library/Extensions/IOAHCIFamily.kext/Contents/PlugIns/IOAHCIBlockStorage.kext/Contents/Info.plist.bak
```
* Next, run the following command in a Terminal window to _inject_ the new option:
```
sudo plutil -replace 'IOKitPersonalities' -json '{"AppleAHCIDiskDriver":{"IOProviderClass":"IOAHCIDevice","IOClass":"AppleAHCIDiskDriver","CFBundleIdentifier":"com.apple.iokit.IOAHCIBlockStorage","Force Data Set Management":true}}' /System/Library/Extensions/IOAHCIFamily.kext/Contents/PlugIns/IOAHCIBlockStorage.kext/Contents/Info.plist
```
* Lastly, _rebuild the kext cache_ so that the modification takes effect:
```
sudo touch /System/Library/Extensions
sudo kextcache -u /
```
* The above command should say "kext-dev-mode allowing invalid signature -67030 0xFFFFFFFFFFFEFA2A for kext IOAHCIBlockStorage.kext".
* _Restart_ the machine and enjoy your official Apple-blessed TRIM support on Yosemite.
* If you ever want to _disable_ TRIM again, just run these two commands and reboot the machine:
```
sudo mv -f /System/Library/Extensions/IOAHCIFamily.kext/Contents/PlugIns/IOAHCIBlockStorage.kext/Contents/Info.plist.bak /System/Library/Extensions/IOAHCIFamily.kext/Contents/PlugIns/IOAHCIBlockStorage.kext/Contents/Info.plist
sudo touch /System/Library/Extensions
```
* Remember that since we've edited IOAHCIBlockStorage.kext, any system updates that replace the kext will remove TRIM again. That problem, and many others, are _completely avoided_ by using the better methods above.
