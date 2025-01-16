
# THIS IS A COPY OF THE ORIGINAL README FOR CLI-VISUALIZER, VIS. 

    ONLY FOR INFO TO USE AS GUIDE FOR BASIC SETUPS.
    *(Installation part is removed from readme)*

# [ Installation from Arch (AUR). ]( https://aur.archlinux.org/packages/cli-visualizer-git)

  - [Setup](#setup)
    - [MPD Setup](#mpd-setup)
    - [ALSA Setup](#alsa-setup)
      - [ALSA with dmix](#alsa-with-dmix)
    - [Pulse Audio Setup (Easy)](#pulse-audio-setup-easy)
  - [Usage](#usage)
  - [Configuration](#configuration)
    - [Reloading Config](#reloading-config)
    - [Colors](#colors)
      - [RGB colors](#rgb-colors)
      - [Color indexes](#color-indexes)
      - [Color names](#color-names)
    - [Spectrum](#spectrum)
      - [Smoothing](#smoothing)
        - [Sgs Smoothing](#sgs-smoothing)
        - [MonsterCat Smoothing](#monstercat-smoothing)
        - [No Smoothing](#no-smoothing)
      - [Falloff](#falloff)
      - [Spectrum Appearance](#spectrum-appearance)
    - [Full configuration example](#full-configuration-example)
  - [Trouble Shooting](#trouble-shooting)
        - [vis hangs with no output](#vis-hangs-with-no-output)


## Setup

At least one of the following needs to be configured (and linked in the config): mpd, alsa and pulseaudio

### MPD Setup

The visualizer needs to use mpd's fifo output file to read in the audio stream. To do this, add the following lines to your mpd config file.


	audio_output {
    	type                    "fifo"
    	name                    "my_fifo"
    	path                    "/tmp/mpd.fifo"
    	format                  "44100:16:2"
	}

If you have any sync issues with the audio where the visualizer either get ahead or behind the music, try lowering the audio buffer in your mpd config.

	audio_output {
        ...
        #this sets the buffer time to 50,000 microseconds
        buffer_time     "50000"
	}

### ALSA Setup

WARNING: alsa support is very very experimental. It is very possible that it will be completely broken on various systems and could cause weird/bad behaviour.

Similar to the MPD setup, the visualizer needs to use a fifo output file to read in the alsa audio stream. To do this, add the following lines to your alsa config file, usually at `/etc/asound.conf`. If the file does not exist create it under `/etc/asound.conf`. 

    pcm.!default {
        type file               # File PCM
        slave.pcm "hw:0,0"      # This should match the playback device at /proc/asound/devices
        file "|safe_fifo /tmp/audio" #safe_fifo will be in the cli-visualizer/bin/safe_fifo directory
        format raw              # File format ("raw" or "wav")
        perm 0666               # Output file permission (octal, def. 0600)
    }

Next change the visualizer config `~/.config/vis/config` to point to the new fifo file

    mpd.fifo.path=/tmp/audio

A normal fifo file can not be used since otherwise no sound would work unless the fifo file is being read. This effectively means that no sound would be played if the visualizer is not running. To get around this issue a helper program `safe_fifo` is used. The `safe_fifo` program is essentially a non-blocking fifo file, it takes `stdin` and writes it to a fifo files given as the first parameter. If the fifo buffer is full, it clears the buffer and writes again.

Note that alsa support is still very experimental. There are a couple of caveats with this approach. Firstly `safe_fifo` must in a location alsa can find it. If you are building from source it us under `cli-visualizer/bin/safe_fifo`. Secondly `slave.pcm` must match whatever your alsa playback device is.

#### ALSA with dmix

On sound cards that do not support hardware level mixing, alsa uses dmix to allow playback from multiple applications at once. In order to make this work with the visualizer a dmixer plugin needs to be defined in `/etc/asound.conf` and the visualizer pcm set to use `dmixer` instead of the hardware device directly. This configuration might will change slightly depending on the system. 

This is an example `asound.conf` for Intel HD Audio.

    pcm.dmixer { 
        type dmix 
        ipc_key 1024
        ipc_key_add_uid false
        ipc_perm 0666            # mixing for all users
        slave { 
            pcm "hw:0,0" 
            period_time 0 
            period_size 1024 
            buffer_size 8192
            rate 44100
        }
        bindings { 
            0 0 
            1 1 
        } 
    } 

    pcm.dsp0 { 
        type plug 
        slave.pcm "dmixer" 
    } 

    pcm.!default { 
        type plug 
        slave.pcm "dmixer" 
    } 

    pcm.default { 
       type plug 
       slave.pcm "dmixer" 
    } 

    ctl.mixer0 { 
        type hw 
        card 0 
    }

    pcm.!default {
        type file               # File PCM
        slave.pcm "dmixer"      # Use the dmixer plugin as the slave pcm
        file "|safe_fifo /tmp/audio"
        format raw              # File format ("raw" or "wav")
        perm 0666               # Output file permission (octal, def. 0600)
    }

### Pulse Audio Setup (Easy)

Pulse audio should be the easiest to setup out of all the options. In order for this to work pulseaudio must be installed and vis must be built with pulseaudio enabled. To build with pulseaudio support run `make ENABLE_PULSE=1`.

To enable pulse audio in the vis config set audio sources to pulse with

    audio.sources=pulse

If this does not work, then vis has most likely not guess the correct sink to use. Try switching the pulseaudio source vis uses. A list can be found by running the command ```pactl info | grep "Source"```. 
The correct pulseaudio source can then be set with

    audio.pulse.source=0

## Usage

Start with

	vis


## Configuration

### Reloading Config

The config can be reload while `vis` is running by either pressing the `r` key or by sending the `USR1` signal to `vis`. Sending the `USR1` signal can be done with `killall -USR1 vis`, this is useful if you want to dynamically reload colors from a script.

### Colors

The display colors and their order can be changed by switching the color scheme in the config under `colors.scheme`.
The color scheme must be defined at `~/.config/vis/colors/color_scheme` directory. There are three different ways to specific a color, by name, by hex number, and by index.

vis does not override or change any of the terminal colors. All colors will be influenced by whatever terminal settings are set by the users terminal. Usually these colors are specified in `.Xdefaults`. There are three different ways to specific a color, by name, by hex number, and by index.

#### RGB colors

The color scheme can be defined in two ways with RGB values or by index. The RGB values are defined as a hex color code, they are useful for creating a 256 bit color scheme.
The displayed color will not be the true color, but instead an approximation of the color based on 256-bit terminal colors.

RGB color scheme example

    #4040ff
    #2c56fc
    #2a59fc
    #1180ed
    #04a6d5
    #02abd1
    #03d2aa
    #04d6a5
    #12ee7f
    #2bfc58
    #2dfc55
    #56fc2d

#### Color indexes

The second way to define a color scheme is by the color index. Specifically this is the exact color index used by ncurses.
Color indexes are useful for creating a visualizer that matches the terminal's set color scheme.

Color index color scheme example

    4
    12
    6
    14
    2
    10
    11
    3
    5
    1
    13
    9
    7
    15
    0



All the basic 16 terminal colors from 0-16 in-order. The spectrum colors can be ordered in any you want, this example was done in order to show all colors.

<br><br>


#### Color names

The third way to define a color scheme is by the color name. Only 8 color names are supported: black, blue, cyan, green, yellow, red, magenta, white.
Note that these 8 colors are the basic terminal colors and are often set by a terminal's color scheme. This means that the color name might not match the color shown since the terminal theme might change it.
For example, dark themes often set `white` to something dark since `white` is usually the default color for the terminal background.



A color scheme with only one color `blue`.

<br><br>


### Spectrum

The spectrum visualizer allows for many different configuration options.

#### Smoothing

There are three different smoothing modes, monstercat, sgs, none.

    #Available smoothing options are monstercat, sgs, none.
    visualizer.spectrum.smoothing.mode=sgs

##### Sgs Smoothing

SGS smoothing [Savitzky-Golay filter](https://en.wikipedia.org/wiki/Savitzky%E2%80%93Golay_filter). There are a couple of options for sgs smoothing.

The first option is controlling the number of smoothing passes done when rendering the bars. Increasing this number with make the spectrum smoother. This is set with `visualizer.sgs.smoothing.passes`.

The second option is the number of neighbors to look at when smoothing. This is set with `visualizer.sgs.smoothing.points`. This should always be an odd number.
A larger number will generally increase the smoothing since more neighbors will be looked at.



Default sgs smoothing.

<br><br>



Sgs smoothing with number of passes set to `5`.

##### MonsterCat Smoothing

Monster cat smoothing is inspired by the monster cat youtube channel (https://www.youtube.com/user/MonstercatMedia). To control the amount of smoothing for montercat use `visualizer.monstercat.smoothing.factor`. The default smoothing factor for monstercat is `1.5`. Increase the smoothing factor by a lot could hurt performance.




##### No Smoothing

Smoothing can be completely turned off by setting the smoothing option to `none`.

    visualizer.spectrum.smoothing.mode=none

Spectrum with smoothing off




#### Falloff

This configures the falloff effect on the spectrum visualizer. This effect creates a slow fall in bar height. Available falloff options are `fill,top,none`. The default falloff option is `fill`.

    visualizer.spectrum.falloff.mode=fill

With the `top` setting the falloff effect is only applied to the top character in the bar. This creates a gap between the main bar and the top falloff character.



top falloff effect with the spectrum character set to `#`.
<br><br>

The `fill` option leaves no gaps between the very top character and the rest of the bar.




fill falloff effect with the spectrum character set to `#`.

<br><br>

The `none` option removing the falloff effect entirely..



Falloff effect removed with the spectrum character set to `#`.

#### Spectrum Appearance

Certain aspects of the spectrum appearance can be controlled through the config.

The bar width can be controlled to make it wider or narrower. The default is `2`.

    visualizer.spectrum.bar.width=2



Bar width set to `5`.

<br><br>

The spacing between bars can also be controlled to make it wider or narrower. The default is `1`.

    visualizer.spectrum.bar.spacing=1



Bar spacing set to `0`.

<br><br>

The margin widths of the spectrum visualizer can also be controlled. The margins are set in percent of total screen for spectrum visualizer. All margin percentages default to `0.0`.

    visualizer.spectrum.top.margin=0.0
    visualizer.spectrum.bottom.margin=0.0
    visualizer.spectrum.right.margin=0.0
    visualizer.spectrum.left.margin=0.0



Spectrum with the top margin set to `0.30` and the left and right margins set to `0.10`.

<br><br>

Normally the bars are ordered so that the lowest frequencies are handled by the left most bars and the highest frequencies by the right most bars.
The reversed option gives the option to reverse this so that the highest frequencies come first.

    visualizer.spectrum.reversed=true



Spectrum with reverse set to `true`.

### Full configuration example

```
    #Refresh rate of the visualizers. A really high refresh rate may cause screen tearing. Default is 20.
    visualizer.fps=20

    #Defaults to "/tmp/mpd.fifo"
    mpd.fifo.path

    #If set to false the visualizers will use mono mode instead of stereo. Some visualizers will
    #behave differently when mono is enabled. For example, spectrum show two sets of bars.
    audio.stereo.enabled=false

    #Specifies how often the visualizer will change in seconds. 0 means do not rotate. Default is 0.
    visualizer.rotation.secs=10

    #Configures the samples rate and the cutoff frequencies.
    audio.sampling.frequency=44100
    audio.low.cutoff.frequency=22050
    audio.high.cutoff.frequency=30


    #Configures the visualizers and the order they are in. Available visualizers are spectrum,lorenz,ellipse.
    #Defaults to spectrum,ellipse,lorenz
    visualizers=spectrum,ellipse,lorenz


    #Configures what character the spectrum visualizer will use. Specifying a space (e.g " ") means the
    #background will be colored instead of the character. Defaults to " ".
    visualizer.spectrum.character=#

    #Spectrum bar width. Defaults to 1.
    visualizer.spectrum.bar.width=2

    #The amount of space between each bar in the spectrum visualizer. Defaults to 1. It's possible to set this to
    #zero to have no space between bars
    visualizer.spectrum.bar.spacing=1

    #Available smoothing options are monstercat, sgs, none. Defaults to sgs.
    visualizer.spectrum.smoothing.mode=monstercat

    #This configures the falloff effect on the spectrum visualizer. Available falloff options are fill,top,none.
    #Defaults to "fill"
    visualizer.spectrum.falloff.mode=fill

    #Configures how fast the falloff character falls. This is an exponential falloff so values usually look
    #best 0.9+ and small changes in this value can have a large effect. Defaults to 0.95
    visualizer.spectrum.falloff.weight=0.95

    #Margins in percent of total screen for spectrum visualizer. All margins default to 0
    visualizer.spectrum.top.margin=0.0
    visualizer.spectrum.bottom.margin=0.0
    visualizer.spectrum.right.margin=0.0
    visualizer.spectrum.left.margin=0.0

    #Reverses the direction of the spectrum so that high freqs are first and low freqs last. Defaults to false.
    visualizer.spectrum.reversed=false

    #Sets the audio sources to use. Currently available ones are "mpd" and "alsa"Sets the audio sources to use.
    #Currently available ones are "mpd", "pulse", and "alsa". Defaults to "mpd".
    audio.sources=pulse

    ##vis tries to find the correct pulseaudio sink, however this will not work on all systems.
    #If pulse audio is not working with vis try switching the audio source. A list can be found by running the
    #command pacmd list-sinks  | grep -e 'name:'  -e 'index'
    audio.pulse.source=0

    #This configures the sgs smoothing effect on the spectrum visualizer. More points spreads out the smoothing
    #effect and increasing passes runs the smoother multiple times on reach run. Defaults are points=3 and passes=2.
    visualizer.sgs.smoothing.points=3
    visualizer.sgs.smoothing.passes=2


    #Configures what character the ellipse visualizer will use. Specifying a space (e.g " ") means the
    #background will be colored instead of the character. Defaults to " ".
    visualizer.ellipse.character=#

    #The radius of each color ring in the ellipse visualizer. Defaults to 2.
    visualizer.ellipse.radius=2

    #Specifies the color scheme. The color scheme must be in ~/.config/vis/colors/ directory. Default is "colors"
    colors.scheme=rainbow

```

## Trouble Shooting

#### vis hangs with no output

It is possible there is a naming conflict with an existing BSD tool `vis`. Try running `vis -h`. If the output looks something like

    mac ❯❯❯ vis -h
    vis: illegal option -- h
    usage: vis [-cbflnostw] [-F foldwidth] [file ...]

then there is a naming conflict with cli-visualizer. To fix this issue, put `/usr/local/bin` before `/usr/bin/` in `$PATH`.

