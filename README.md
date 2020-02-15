# Xdpi

A flexible display configuration manager for \*nix systems, written in pure POSIX sh

## Motivation

Typically, \*nix systems do not provide much in terms of the connection and management of external displays. While connecting a display and running the appropriate `xrandr` command will allow the X server to output to the monitor with the specified resolution and refresh rate, there are still many application-level problems that are left unresolved. For example, if the configuration file of an application includes a DPI specification, or this application resolves the appropriate DPI via the X server resources database (e.g. specified in `$HOME/.Xresources`), this application will continue using the DPI spec assumed for the previous display (assuming no manual intervention). If the DPI of the new display differs from that of the previous display, this will result in the application either being much too large, or much too small. To make matters worse, if an application configuration requires a specification for the monitor itself (e.g. `eDP1`, `DP2`, `HDMI3`, etc.) the application in question may not even display on the new monitor without the appropriate configuration change.

To alleviate these issues, I present `xdpi`. `xdpi` is a tool to create, save, and manage configurations for any display that may be commonly (dis)connected (e.g. laptop + external display). These configurations describe all the changes that must be made as well as commands that must be run when connecting to the specified display. Edits to be made on application configuration files are saved in a generalized `diff`-like format, such that <i>only the changes</i> to these configurations must be specified. Edits to the X server resource database are saved in a specialized `x.dpi` file which is automatically included by your `.Xresources` file. These files are unified under a single directory per `xdpi` configuration, and each `xdpi` configuration is located under a single directory located in `$XDG_CONFIG_HOME`.

Upon execution of `xdpi -s [configuration]`, updates are made and applications reloaded. To provide even more flexibility, the user also has the option of creating a shell script for each `xdpi` configuration that will run <i>prior</i> to these updates taking place. The reason for this ordering and an example of its efficacity is described below. Allowing the execution of this user-created script allows for near-limitless flexibility. Common use cases I've found to leverage this are to use this script to execute the appropriate `xrandr` command to enable new display(s)/disable others, setting the background of the new display(s) with your favorite wallpaper utility (e.g. `feh`), and even setting the PulseAudio profile/sink if for example your monitor has built-in speakers. An example of this may be found near the bottom of the README.

## Configuration Tree

Every `xdpi` configuration is located in `$XDG_CONFIG_HOME/xdpi/configs` and assumes the following format:

```
configs
└── my-config
    ├── my-config.dpi
    ├── my-config.sh
    └── diffs
        ├── application-1.diff
        ├── application-2.diff
        ┋
        └── application-n.diff
```

When creating a new configuration via `xdpi -n [configuration]`, a "blank-slate" directory is created under the configuration name specified. Following the example above, executing `xdpi -n my-config` would create an empty `my-config` directory, under which would be the files `my-config.dpi` and `my-config.sh`, as well as an empty `diffs` directory.

## Order of Execution

When switching configurations, the order of execution is as follows: 

- Ensure `$HOME/.Xresources` includes `$XDG_CONFIG_HOME/xdpi/x.dpi`
- Copy the user-created `.dpi` file to `$XDG_CONFIG_HOME/xdpi/x.dpi`
- Merge `$HOME/.Xresources` into the X server resource database (via `xrdb`)
- Execute the user-created `.sh` script
- Loop through `.diff` files and apply changes to the specified application configs
- Reload applications specified in `$PROC_LIST`

Note: user shell script execution being before the `diffs` loop is <i>purposeful</i> as it allows for the possibility of <i>dynamic edits</i> to various `.diff` files prior to their execution. This allows for managing configuration changes that aren't necessarily static.

## `.dpi` File Specification

Since the configuration `.dpi` file is intended to be included in your `$HOME/.Xresources` via a `cpp` preprocessor, this file must follow the exact same spec as an X resource file.

### Example

```
! Xft
Xft.dpi: 96
Xft.autohint: 0
Xft.antialias: 1
Xft.hinting: true
Xft.hintstyle: hintslight
Xft.rgba: rgb
Xft.lcdfilter: lcddefault

! Font
*.font: Ubuntu Mono:pixelsize=16
```

## `.diff` File Specification

`xdpi` `.diff` files follow a generalized format similar to regular `.diff` files. The first line should start with a `#` character, followed by the path to the application configuration file you wish to edit. This is the <i>only</i> line that should contain a `#` character. After this, all lines beginning with the `>` character should contain a regular expression of the line in that configuration file you wish to change (specifically, `sed` regex), and all lines beginning with `<` should contain what you wish to replace it with. All other lines are ignored (but may be used to make things look pretty!). Every `>` regular expression <i>must</i> match a `<` line, otherwise it will be ignored.

### Example

```diff
# /home/gs/.config/polybar/config

< monitor = ".*"
< dpi = .*
< height = .*
< sink = .*
---
> monitor = "DP1"
> dpi = 96
> height = 38
> sink = alsa_output\.pci-0000_00_1f\.3\.hdmi-stereo
```

## Shell Script Example

It was previously mentioned that the reasoning for the shell script being executed prior to the application of the `.diff` files is due to the fact that this allows for dynamically editing `.diff` files where the configuration change necessary is not static. An example of this may be seen below. When connecting to a display with dedicated speakers, PulseAudio will choose an audio sink index for the device. Unforunately, the index chosen will not necessarily be the same on every reconnect. Since `pactl` requires a specification of the audio sink index for the `set-sink-volume` and `set-sink-mute` commands (whose execution is commonly bound to `XF86AudioRaiseVolume`, `XF86AudioLowerVolume`, and `XF86AudioMute` in various WM configs (e.g. `i3`) to allow media key functionality), the WM `.diff` file (e.g. `i3.diff`) must be edited such that the new sink index index matches that of the monitor audio sink index.

```sh
#!/bin/sh

XDPI_CONF_DIR="$XDG_CONFIG_HOME/xdpi/configs"
CURR_CONF_DIR="$XDPI_CONF_DIR/uw"
i3_DIFF_PATH="$CURR_CONF_DIR/diffs/i3.diff"

# Get pulseaudio card index
AUDIO_CARD_INDEX=$(pacmd list-cards | grep "index" | sed "s/^.*index:\s*//")

# Turn off eDP1 display, turn on DP1 display
xrandr --output eDP1 --off --output DP1 --mode 3440x1440 --pos 0x0 --rotate normal
--output DP2 --off --output HDMI1 --off --output VIRTUAL1 --off

# Change bg to fit ultrawide display
feh --bg-fill $HOME/Pictures/3440x1440/jupiter.png

# Change pulseaudio profile to hdmi-stereo
pacmd set-card-profile $AUDIO_CARD_INDEX output:hdmi-stereo+input:analog-stereo

# Get new audio sink index
AUDIO_SINK_INDEX=$(pacmd list-sinks | grep "index" | sed "s/^.*index:\s*//")

# Ensure i3.diff audio sink is accurate before applied so media keys work properly
sed -i "s/^>\s*set-sink-volume.*$/> set-sink-volume $AUDIO_SINK_INDEX/" "$i3_DIFF_PATH"
sed -i "s/^>\s*set-sink-mute.*$/> set-sink-mute $AUDIO_SINK_INDEX/" "$i3_DIFF_PATH"
```

## Contributing and Testing

Contributions and testing on various environments (distros/WMs/DEs/bars etc.) is welcome and encouraged! If you find a bug, want support for an application not already covered (i.e. does not reload automatically), or find a better way of going about things, please feel free to send me a message or issue a pull request.
