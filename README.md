# Open Media Transport Plugin

omtplugin is a cross platform Open Media Transport plugin for OBS (https://obsproject.com)

> [!IMPORTANT]
> **This is a patched fork, not the official plugin.** The official plugin is at
> [openmediatransport/omtplugin](https://github.com/openmediatransport/omtplugin).
> This fork exists to carry audio fixes that upstream decided not to take.
> It is maintained for our own production rigs, with no support and no release schedule.

## About this fork

### Why it exists

The official plugin treats all OMT audio as stereo, in both directions. That breaks mono
sources and drops every channel after the second on surround or multitrack feeds. We
offered the receive fix upstream
([openmediatransport/omtplugin#9](https://github.com/openmediatransport/omtplugin/pull/9)).
The maintainers don't consider this a problem, so the PR was closed and the fixes live
here instead.

### What's patched

All patches are on the `fix/output-audio-channels` branch. `master` tracks upstream unchanged.

| Area | Upstream behaviour | Patched behaviour | Commit |
|---|---|---|---|
| OMT Source (receive) | Always tells OBS the audio is stereo and passes two planes. A **mono** source makes OBS read the second plane past the end of the buffer (garbage audio, possible crash). Channels 3 and up are dropped. | Uses the frame's channel count and the matching OBS speaker layout: mono, stereo, 2.1, 4.0, 4.1, 5.1, 7.1. Seven channels play as 7.1 with a silent eighth; more than eight are cut to the first eight. Frames shorter than their stated channel count are skipped. | `ee34bee` |
| OMT Output (send) | Always declares 2 channels, even when OBS is set to 5.1 or 7.1, and sends one pointer on the assumption that OBS's channel buffers sit back to back. | Sends the channel count from OBS's speaker layout (Settings > Audio > Channels). Copies each channel into one buffer before sending, under the same lock that stops the output. | `35bcae6` |
| OMT Source color space | Choosing **BT709_P010** falls through to OBS's default color space because the BT.709 check tests `BT709` twice. | BT709_P010 uses the BT.709 color space, as BT709 does. | `eb3f032` |
| OMT Source address | The Source box is a fixed list of sources that discovery has found. A source on another subnet, or one discovery hasn't listed yet, can't be picked. | The Source box is editable: pick from the list or type an address. A typed address connects 750 ms after you stop typing, so the receiver isn't rebuilt on every keystroke. Leading and trailing spaces are trimmed. | `0d47cc5` |
| OMT Source, clearing the source | Choosing the blank entry disposes the receiver but keeps using it afterwards. | The disposed receiver is dropped, so clearing the source is safe. | `0d47cc5` |
| OMT Output quality | Always sends at OMT's Default quality. Tools > OMT Output Settings has only Enabled and Name. | OMT Output Settings has a **Quality** choice: Default, Low, Medium or High. It's saved with the other output settings, and configs saved before this change keep Default. Changing it restarts the output. | see `git log` |

Everything else (video formats, HDR PQ and HLG, tally, discovery) is the same as upstream.

### Things to know

- **4-channel sources** map to OBS 4.0 (front left, front right, front centre, rear centre).
  That suits surround, but not four separate mics.
- **Receivers may now get 6 or 8 channels.** With OBS set to 5.1 or 7.1, the OMT output sends
  every channel. Check that your receivers can handle that, or set OBS to Stereo.
- **Not yet tested on live rigs.** Both patches build cleanly as a native Windows plugin, but they
  still need testing with mono, stereo, 4, 6 and 8 channel sources.

### Building

There are no prebuilt releases for this fork. Build from the patched branch:

```
git clone -b fix/output-audio-channels https://github.com/chillico/omtplugin
```

Then follow the build steps below. The project expects `libomtnet` to be cloned and built next to
this repo (`../libomtnet/bin/Release/netstandard2.0/libomtnet.dll`).

On Windows, the native build runs `vswhere.exe`. If the link step fails with
`'vswhere.exe' is not recognized`, add `C:\Program Files (x86)\Microsoft Visual Studio\Installer`
to `PATH` and build again.

### Staying in sync

Pull upstream into `master`, then rebase `fix/output-audio-channels` onto it. Re-test audio after
each rebase, because both patches touch the receive and output code paths.

---

*The rest of this README is the upstream README, unchanged. Its Download link points to the
official releases, which don't include these patches.*

## Download

https://github.com/openmediatransport/omtplugin/releases

## Features

Send and receive high quality audio/video between OBS and other OMT compatible devices over a local network!

For more information about Open Media Transport in general, see the README here:
https://github.com/openmediatransport/

### OMT Source

Receive audio/video from OMT sources on the network.

Supports HDR workflows using the Color Space dropdown in the OMT Source properties.

### OMT Output

Send audio/video from the main OBS output.

This can be enabled from the Tools - OMT Output Settings menu.

This supports HDR OBS workflows and will send 10bit+ when P010 or P216 is enabled in OBS settings.

## Requirements

OBS 31 or higher

## Installation

### Windows

1. Download the Windows binaries from Releases on GitHub
2. Copy omtplugin.dll and libvmx.dll into the following folder:

```
C:\ProgramData\obs-studio\plugins\omtplugin\bin\64bit
```

This folder will need to be created if it does not already exist.

**Note:** Replace C: if Windows has been installed to a different drive.

### MacOS

1. Download the MacOS pkg from Releases on GitHub
2. Open the package and follow the instructions to install


### Linux

The install instructions below are based on Linux Distributions using the **APT** package manager.

1. Update the package manager

```
sudo apt update
```

2. Install dotnet 8

```
curl -sSL https://dot.net/v1/dotnet-install.sh | bash /dev/stdin --channel 8.0

echo 'export DOTNET_ROOT=$HOME/.dotnet' >> ~/.bashrc
echo 'export PATH=$PATH:$HOME/.dotnet' >> ~/.bashrc
source ~/.bashrc
```

3. Install Clang and Git

```
sudo apt install clang
sudo apt install git
```

4. Download the required repositories from Github into the home directory.

```
cd ~/
git clone https://github.com/openmediatransport/libvmx
git clone https://github.com/openmediatransport/libomtnet
git clone https://github.com/openmediatransport/omtplugin
```
5. Build libvmx 

```
cd ~/libvmx/build
chmod 755 buildlinuxx64.sh
./buildlinuxx64.sh
```
6. Build libomtnet 

```
cd ~/libomtnet/build
chmod 755 buildall.sh
./buildall.sh
```

7. Build omtplugin

```
cd ~/omtplugin/build
chmod 755 buildlinuxx64.sh
./buildlinuxx64.sh
```

8. Copy libraries into OBS plugins directory

```
sudo cp ~/omtplugin/bin/Release/net8.0/linux-x64/native/omtplugin.so /usr/lib/x86_64-linux-gnu/obs-plugins/
sudo cp ~/libvmx/build/libvmx.so /usr/lib/x86_64-linux-gnu/obs-plugins/
```
