# rsudp
![Raspberry Shake logo](doc_imgs/raspbery-shake-logo-2x.png)

### Tools for receiving and interacting with Raspberry Shake UDP data
*Written by Ian Nesbitt (@iannesbitt) and Richard Boaz (@ivor) for @osop*  

`rsudp` is a tool for receiving and interacting with UDP data sent from a Raspberry Shake seismograph. It contains four main features:
1. Print - a debugging tool to output raw UDP output to the command line
2. Alarm - an earthquake/sudden motion alert configured to run some code in the event of a recursive STA/LTA alarm trigger, complete with bandpass filter capability
3. Writer - a miniSEED writer
4. Plot - a live-plotting routine to display data as it arrives on the port

`rsudp` is written in Python but requires no coding knowledge to run. Simply go to your Shake's web front end, point a UDP data cast at your computer's local IP address, and watch as the data rolls in.

## Notes about `rsudp`

**Note: The port you send data to must be open on the receiving end.** In Windows, this may mean clicking "allow" on a firewall popup. On most other machines, the port you send UDP data to (8888 or 18001 are common choices) must be open to UDP traffic.

Generally, if you are sending data inside a local network, there will not be any router firewall to pass data through. If you are sending data to another network, you will almost certainly have to forward data through a firewall in order to receive it. Raspberry Shake cannot help you figure out how to set up your router to do this. Contact your ISP or network administrator, or consult your router's manual for help setting up port forwarding.

**Note: this program has not been tested to run on the Raspberry Shake itself.** Raspberry Shake is not liable to provide support for any strange Shake behavior should you choose to do this. This program is intended to run on a separate RPi or workstation, and for the Raspberry Shake to cast data to that computer.


## Installation

### On Linux & MacOS

A UNIX installer script is available at `unix-install-rsudp.sh`. This script checks whether or not you have Anaconda installed, then downloads and installs it if need be. This script has been tested on both `x86_64` and `armv7l` architectures (meaning that it can run on your home computer or a Raspberry Pi) and will download the appropriate Anaconda distribution, set up a virtual Python environment, and leave you ready to run the program. To install using this method:

```bash
$ bash unix-install-rsudp.sh
```

Once you've done this, your conda environment will be available by typing:
```bash
$ conda activate rsudp
```
Your prompt should now look like the following:
```bash
(rsudp) $
```
From here, you can begin using the program.

**Note: the installer script will pause partway through to ask if you would like to make the `conda` command executable by default. This is done by appending the line below to your `~/.bashrc` file.** This is generally harmless, but if you have a specific objection to it, hitting any key other than "y" will cause the script to skip this step. You will have to manually run the `conda` executable in this case, however. If you choose to do it manually later, the line appended to the end of `~/.bashrc` is the following (architecture-dependent):

On x86 systems:
```bash
. $HOME/miniconda3/etc/profile.d/conda.sh
```
or on ARMv7 architecture with Raspbian OS:
```bash
. $HOME/berryconda3/etc/profile.d/conda.sh
```
*Note: You can run `uname -m` to check your computer's architecture.*

where `$HOME` is the home directory of the current user.

### On Windows

1. Download and install [Anaconda](https://www.anaconda.com/distribution/#windows) or [Miniconda](https://docs.conda.io/en/latest/miniconda.html).
2. Open an Anaconda Prompt.
3. Execute the following lines of code:

```bash
conda config --append channels conda-forge
conda create -n rsudp python=3 matplotlib=3.1.1 numpy=1.16.4 future scipy lxml sqlalchemy obspy
conda activate rsudp
pip install rsudp
```
## Using this software

First, you will need a data cast (formerly known as a UDP stream) pointed at an open port on the computer you plan to run this on. By default this port is 8888. To list the default settings used by the program, type `shake_client -d` to print the default settings. To dump these settings to a file for modification, type `shake_client -d > rsudp_settings.json`.

After modifying the settings file to your liking, type `shake_client -s rsudp_settings.json` to run.

**Note:** This library can only handle incoming data from one Shake per port. If for some reason more than one Shake is sending data to the port, the software will only process data coming from the IP of the first Shake it sees sending data.

## Settings

By default, the settings are as follows:

```json
{
"settings": {
    "port": 8888,
    "station": "Z0000"},
"printdata": {
    "enabled": false},
"write": {
    "enabled": false,
    "outdir": "/home/pi",
    "channels": "all"},
"plot": {
    "enabled": true,
    "duration": 30,
    "spectrogram": false,
    "fullscreen": false,
    "eq_screenshots": false,
    "channels": ["HZ", "HDF"],
    "deconvolve": false,
    "units": "ACC"},
"forward": {
    "enabled": false,
    "address": "192.168.1.254",
    "port": 8888,
    "channels": ["all"]},
"alert": {
    "enabled": true,
    "highpass": 0,
    "lowpass": 50,
    "deconvolve": false,
    "units": "ACC",
    "sta": 6,
    "lta": 30,
    "threshold": 1.7,
    "reset": 1.6,
    "exec": "eqAlert",
    "channel": "HZ",
    "alertsound": false,
    "mp3file": "rs_sounds/doorbell.mp3",
    "win_override": false,
    "debug": false}
}
```

- **`settings`** contains `"port"` and `"station"` defaults. Change these if you are receiving the data at a different port than 8888, or if you would like to set your station name.

- **`printdata`** controls the debug module, which simply prints the Shake data packets it receives to the command line. Change `"enabled"` to `true` to activate.

- **`write`** controls a very simple miniSEED writer. Every 10 seconds, seismic data is appended to a file with a descriptive name in the directory specified after `"outdir"`. By default, this directory is `"/home/pi"` which will need to be changed to the location of an existing directory on your machine or it will throw an error. By default, `"all"` channels will be written to their own files. You can change which channels are written by changing this to, for example, `["EHZ", "ENZ"]`, which will write the vertical geophone and accelerometer channels from RS4D output.

- **`plot`** controls the thread containing the plotting algorithm. This module can plot seismogram data from a list of 1-4 Shake channels, and optionally calculate and display a spectrogram alongside each (to do this, set `"spectrogram"` to `true`). By default the `"duration"` in seconds is `30`. The longer the duration, the more time it will take to plot, especially when the spectrogram is enabled. To put this plot into kiosk mode, set `"fullscreen"` to `true`. On a Raspberry Pi 3B+ plotting 600 seconds' worth of data and a spectrogram from one channel, the update frequency is approximately once every 5 seconds, but more powerful processors should be able to keep up with the data rate for larger-than-default `"duration"` values and more than just one channel. The plot will update at most once per second.

  The program will use the Raspberry Shake FDSN service to search for an inventory for the Shake you specify in the `"station"` field. If it successfully finds an inventory, setting `"deconvolve"` to `true` will deconvolve the channels plotted to either `"ACC"` (acceleration in m/s^2), `"VEL"` (velocity in m/s), or `"DISP"` (displacement in m). This means that the Shake must both have the 4.5 Hz geophone distributed by RS, and be forwarding data to the Shake server, in order to properly calculate deconvolution.

  If the alert module is enabled, setting `"eq_screenshots"` to `true` will result in the script saving one PNG figure per alert to the default config location (`.config/rsudp/screenshots` in the user's home folder) when the leading edge of the quake is about 60% of the way across the plot window. This will only occur when the alarm gets triggered, however, so make sure to test your alert settings thoroughly.

- **`alert`** controls the alert module (please see [Disclaimer](#disclaimer) below). The alert module is a fast recursive STA/LTA sudden motion detector that utilizes obspy's [`recursive_sta_lta()`](https://docs.obspy.org/tutorial/code_snippets/trigger_tutorial.html#recursive-sta-lta) function. STA/LTA algorithms calculate a ratio of the short term average of station noise to the long term average. The data can be highpass, lowpass, or bandpass filtered by changing the `"highpass"` and `"lowpass"` parameters from their defaults (0 and 50 respectively). By default, the alert will be calculated on raw count data from the vertical geophone channel (either `"SHZ"` or `"EHZ"`). It will throw an error if there is no Z channel available (i.e. if you have a Raspberry Boom with no geophone). If you have a Boom and still would like to run this module, change the default channel `"HZ"` to `"HDF"`.

  If the STA/LTA ratio goes above a certain value (`"threshold"`), then the module runs a function passed to it. By default, this function is `rsudp.client.eqAlert()` which just outputs some text, and optionally loads and plays an MP3 sound (if `"alertsound"` is `"true"` and you have either `ffmpeg` or `libav` installed). For details on installation of these dependencies, see [this page](https://github.com/jiaaro/pydub#dependencies)). When the ratio goes back below the `"reset"` value, the alarm is reset.
  
  If you have either `ffmpeg` or `libav` installed and would like to play an sound alert when the threshold is exceeded, the software will install several small MP3 files. The `"mp3file"` is `"doorbell"` by default, but there are a few more aggressive alert sounds, including: a three-beep sound `"beeps"`, a sequence of sonar pings `"sonar"`, and a continuous alarm beeping for 5 seconds, `"alarm"`. You can also point the `"mp3file"` field to an MP3 file somewhere in your filesystem. For example, if your username was `pi` and you had a file called `earthquake.mp3` in your Downloads folder, you would specify `"mp3file": "/home/pi/Downloads/earthquake.mp3"`. The program will throw an error if it can't find (or load) the specified MP3 file. It will also alert you if the software dependencies for playback are not installed.
  
  You can also change the `"exec"` field and supply a path to executable Python code to run with the `exec()` function. Be very careful when using the `exec()` function, as it is known to have problems. Notably, it does not check the passed code for errors prior to running. Additionally, if the code takes too long to execute, you could end up losing data packets, so keep it simple (sending a message or a tweet, which should either succeed or time out in a few seconds, is really the intended purpose). In testing, we were able to get the pydub software to play 30 second-long sounds without losing any data packets. Theoretically you could run code that takes longer to process than that, but the issue is that the longer it takes the function to process code, the longer the module will go without processing data from the queue (the queue can hold up to 2048 packets, which for a RS4D works out to 128 seconds' worth of data).

  If you are running Windows and have code you want to pass to the `exec()` feature, Python requires that your newline characters are in the UNIX style (`\n`), not the standard Windows style (`\r\n`). To convert, follow the instructions in one of the answers to this [stackoverflow question](https://stackoverflow.com/questions/17579553/windows-command-to-convert-unix-line-endings). If you're not sure what this means, please read about newline/line ending characters [here](https://en.wikipedia.org/wiki/Newline). If you are certain that your code file has no Windows newlines, you can set `"win_override"` to `true`.

## Disclaimer

**NOTE: It is extremely important that you do not rely on this code to save life or property.** Raspberry Shake is not liable for earthquake detection false positives, false negatives, errors running the Alert module, or any other part of this library; it is meant for hobby and non-professional notification use only. If you need professional software meant to provide warning that saves life or property please contact Raspberry Shake directly or look elsewhere. See sections 16 and 16b of the [License](LICENSE).

## pydub dependencies

If you would like to play sounds when the STA/LTA trigger activates, you will need to take the following steps:

1. [Install the release version of this software](#installation) if you have not already.
2. [Install the dependencies for `pydub`](https://github.com/jiaaro/pydub#dependencies) which are available cross-platform using most package managers. (Linux users can simply type `sudo apt install ffmpeg`, and MacOS users with Homebrew can type `brew install ffmpeg`) but Windows users will have to follow more detailed instructions.
3. Change `"alertsound"` from `false` to `true` in the settings file. (`~/.config/rsudp/rsudp_settings.json` on Mac and Linux, `C:\Program Files\RSHAKE\rsudp\rsudp_settings.json` on Windows)
4. Start the rsudp client by typing `shake_client` or by pointing it at an existing settings file `shake_client -s /path/to/settings.json`
5. Wait for the trigger to warm up, then stomp, jump, or Shake to hear the sound!

# Contributing

Contributions to this project are more than welcome. If you find ways to improve the efficiency of the library or the modules that use it, or come up with cool new modules to share with the community, we are eager to include them (provided, of course, that they are stable and achieve a clearly stated goal).

Since the Producer function passes an `ALARM` queue message when it sees `Alert.alarm=True`, other modules can be programmed to do something when they see this message. This is to help make the addition of other action-based modules easy.

Some ideas for improvements are:
- a more efficient plotting routine
- a module that creates a twitter post when it reads the "ALARM" queue message
- a way to save plot screenshots some time after an alert function trigger, to save and document earthquake seismogram/spectrograms
- GPIO pin interactions (lights, motor control, buzzers, etc.)

