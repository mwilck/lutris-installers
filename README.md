# Installers for lutris

Installer files for [lutris](https://lutris.net/) which can be invoked locally
using `lutris -di "$FILE.yaml"`. I'll also upload them (more precisely, the
`script` section) to the lutris games data base.

# How to write a Lutris installer? Lessons learned

I have a collection of PC game CDs from the 90s and early 2000s. Since quite
some time I've wanted to try these games in Lutris. I'm aware that many of
them are already available via gog.com and other online services. Yet I do
have the original CDs, I've spent a lot of hours with them in the past, so I
want to try to see how far I get with Lutris.

Documentation on writing installers is rather scarce. There's
the official [Writing installers](https://github.com/lutris/lutris/blob/master/docs/installers.rst)
document (which is also shown to you when you try to submit an installer via
the lutris web site), but at least for me, that left a lot of open
questions. I tried to craft my first installers with the help of this document
and a few random installers for other older games. Unfortunately, there's no
way to search for "dosbox installers for CD-ROM games", which might have
turned up powerful templates.

All my experiments were made on openSUSE Leap 15.3, which is a stable
distribution and ships some rather old versions of various tools.

## Dependencies

On my rather fresh and lean Leap 15.3 installation, I had to install
`libopusfile0` and `libSDL2_net-2_0-0` to make the **dosbox** runner from Lutris
work. I also tried using my systems native **dosbox** (0.74.3) which worked
out of the box, but had some issues with screen resolution that I spent a long
time tapering with. It would have been faster to figure out the dependencies
in the first place.

## Midi

Many old games play MIDI, which was supported by hardware synthesizers on old
sound cards SoundBlaster Pro. Most modern sound cards don't include hardware
synthesizers. The net recommends to use a software synthesizer, **FluidSynth**
and **timidity++** are mentioned. The latter hasn't seen any updates for
almost 20 years, so I went for fluidsynth.

On typical modern Linux systems using **pulseaudio**, fluidsynth needs to be
run with the `pulseaudio` audio driver and the `alsa_seq` MIDI driver. It
creates a MIDI port that e.g. dosbox can use for MIDI playback. All else
that's needed is a decent sound font. On openSUSE Leap, the **lutris** package
pulls in the Fluid (R3) SoundFont via the `fluid-soundfont-gm` package (but it
doesn't pull in the `fluidsynth` package, strangely. A full FluidSynth command
line then looks like this:

    usr/bin/fluidsynth -is -a pulseaudio -m alsa_seq /usr/share/sounds/sf2/FluidR3_GM.sf2

I've create this **systemd** unit file
and stored it as `$HOME/.local/share/systemd/user/fluidsynth.service`:

```
[Unit]
Description=FluidSynth Daemon
Documentation=man:fluidsynth(1)
PartOf=sound.target
After=pulseaudio.service
Requires=pulseaudio.socket

[Service]
LimitRTPRIO=50
ExecStart=/usr/bin/fluidsynth -is -a pulseaudio -m alsa_seq /usr/share/sounds/sf2/FluidR3_GM.sf2

[Install]
WantedBy=sound.target
```

The `LimitRTPRIO` directive may need further tuning in system-wide
config files to become effective. It isn't truely necessary on modern systems.

## CD audio (music tracks)

Some DOS games play CD audio. Remember those extra audio cables that went from the
old CD drives to the main board? This can't be easily done from ISO
images. **dosbox** supports it with "CUE/BIN" images, though. They need to be
mounted using the dosbox command `imgmount D $CUE_FILE -t cdrom`. The
`$CUE_FILE` must be created before, and [special care must bese
taken](https://www.dosbox.com/wiki/Cuesheet) to use the [right
cdrdao options](https://github.com/cdrdao/cdrdao) when extracting the image
e.g. with **cdrdao**, otherwise you'll here just noise:

    cdrdao read-cd --read-raw --device /dev/cdrom \
	    --driver generic-mmc:0x20000
        --datafile "$FILE.bin" "$FILE.toc"
	toc2cue "$FILE.toc" "$FILE.cue"

The flag `0x20000` inverts the byte order of the audio stream. Not sure if
this is only necessary on little-endian systems.

## YAML structure

The installers on lutris.net represent only the `script` section of YAML files
that can be used for local testing. The latter need to have a stucture like
this:

```
# The following root elements are required by lutris.
# For installers downloaded from lutris.net, they are entered
# via the web form or (the "slug" fields) auto-generated.
name:      "Die Siedler II (with CD music)"
game_slug: "die-siedler-2-music"
runner:    "dosbox"
slug:      "die-siedler-2-music-installer"
version:   "Die Sieder II (copy CD-ROM to hard drive)"
script:
  # installer goes below here
```

## dosbox

This is how a basic structure of a dosbox game looks like (using dosbox both
for installation and runtime):

```
  game:
     main_file: ...   # for dosbox
  require-binaries: cdrdao, ...   # fail early if tools are missing
  installer:
     # actual installer workflow steps
	 - insert-disc:
       # wait for CD to be inserted
	 - execute:
       # Linux command and args
	 - write_file:
	   # create arbitrary file from YAML
	 - task:
	   # a runner task
	   name: dosexec     # for dosbox
	   config_file:      # dosbox config
	   executable:       # DOS command / batch f
```

### dosbox config files

Config files can be simply created with `write_file` sections. Here's a sample
from "Die Siedler II", creating the full environment for the game:

```
    - write_file:
        file: $GAMEDIR/siedler2.conf
        content: |
          [autoexec]
          mount C $GAMEDIR
          imgmount D $GAMEDIR/CD/SIEDLER2.cue -t cdrom
          C:
          cd \BLUEBYTE\SIEDLER2
          config -securemode
          C:\BLUEBYTE\SIEDLER2\SIEDLER2.EXE dt
          exit
```

Note that this installer is using `imgmount` for the CD-ROM, which it has
converted to a hard disk image before.

### dosexec, "executable", C:, and exiting

I had an issue with `dosexec` tasks which would **not** exit after the DOS
program had finished, despite and `exit:` clause and an `exit` command in the
dosbox config file. I found that I had to set an `executable`; after executing
the latter, dosbox would actually exit. However, `executable` file are run
from drive `C:` always, and dosbox actually switches to the `C:` before
running them, which doesn't work for a setup program on the CD. Swapping `C:`
and `D:` is a very bad idea. So, a minimal batch script must be created under
`C:` and set as `executable`.

