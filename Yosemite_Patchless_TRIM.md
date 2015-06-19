# Yosemite Patchless TRIM Support

Apple started laying the groundwork for official 3rd party TRIM support as early as OS X 10.10.3, when they changed their IOAHCIBlockStorage.kext to understand a new Info.plist option that allows the user to force their driver to enable TRIM on non-Apple SSDs.

Officially, the hidden option is "supposed" to be enabled via their new "trimforce" tool, but all their tool does is copy a new file called "AppleDataSetManagement.kext" into the system folder. That kext in turn just injects the new option into IOAHCIBlockStorage.kext at runtime. That's all good so far, but unfortunately that new tool is only available on OS X El Capitan.

This made me think a bit, and after some testing I devised a way to enable TRIM on OS X Yosemite 10.10.3 or higher -- without requiring any binary patching! Unfortunately we still break the code signature when we edit Info.plist, so we still need to enable kext-dev-mode until El Capitan comes out. (_Update: This drawback is completely avoided if you use the "Even Better Method" below!_)

The advantages of this new Info.plist method over the classic binary patching is that this is the official Apple method and is guaranteed to work as long as you are on 10.10.3 or higher (lower versions do _not_ work).

OS X 10.11 El Capitan users should simply use the new "trimforce" command instead (by disabling Rootless, running "sudo trimforce enable", and re-enabling Rootless). _Hopefully Apple improves trimforce before El Cap's release, so that it no longer requires us to temporarily disable rootless. That seems like an oversight on their part._


## Enabling TRIM Support on OS X 10.10.3+ (Manual Method)

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


## Even Better Method (No kext-dev-mode required!)

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
* Restart your machine. This is the official method used by "trimforce", so you don't have to worry about nvram (kext-dev-mode), binary or Info.plist patches, software updates removing TRIM support, booting to a gray stop sign error, and so on. Everything is properly codesigned by Apple and permanently enables TRIM via the official method.

**Disclaimer**: Enabling TRIM is done at your own peril and may cause data corruption if you're using an old SSD with a buggy TRIM command. You cannot make any claims against Apple or myself if something bad happens with your SSD. If "trimforce" actually arrives in OS X Yosemite 10.10.4, I will _remove_ all of the instructions on this page (but it's _not_ yet available in the latest 10.10.4 beta, 14E36b from June 15th, 2015, so it looks like Apple has forgotten to backport it). The manual kext installation method above will only be required for as long as Apple hasn't released it officially for Yosemite. And if anyone from Apple's legal department wants me to remove the links or all of these instructions, then open an issue on my Github issue tracker and I will take everything down as soon as I see it. I just want to help bring TRIM to people who have been waiting for it for way too long -- and trying to stop people from installing your official kext on earlier OS X versions is not the right move morally. Oh, and please remember to include trimforce in 10.10.4. ;-)
