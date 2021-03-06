Copyright (C) 2007 L. Donnie Smith <donnie.smith@gatech.edu>

This program is free software; you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation; either version 2 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program; if not, write to the Free Software
Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA

OVERVIEW
--------

wminput is an Linux event, mouse, and joystick driver for the wiimote using the
uinput system. It supports assigning key/button symbols to buttons on the
wiimote, nunchuk, and classic controller, and axes symbols to wiimote axes
including direction pads, "analog" sticks, and "analog" shoulder buttons.
Furthermore, it provides a plugin interface through which more advanced
functionality can be implemented, such as accelerometer and ir calculations.
Plugins can provide button-type events and axes.

EXECUTION
---------

```
wminput [-h] [-w] [-c config] [bdaddr]
```

-h: print usage and exit

-w: on startup, wait (without timing out) until a wiimote is found

-c config: specifies the configuration file to load. "default" is the default
configuration file (on install, a symlink is created to the `acc_ptr`
configuration. The search directories are specified below in the section on
configuration files, or an absolute or relative pathname may be given.

bdaddr: specifies the bluetooth device address of the wiimote. If unspecified,
the environment variable `WIIMOTE_BDADDR` is used. If this variable does not
exist, a connection is made with the first wiimote found.


CONFIGURATION REQUIREMENTS
--------------------------

uinput kernel support is required. joydev and evdev are required for joystick
and event interfaces, respectively. See
[HOWTO Compile a Kernel Manually](http://gentoo-wiki.com/HOWTO_Compile_a_Kernel_Manually)
for information on kernel compilation.

By default, some (most? all?) udev configurations set up a uinput device file
readable only by root. Using wminput as a user other than root requires udev to
change the permissions on uinput. Place the following line in a file in
`/etc/udev/rules.d` (see the documentation for your distro for the recommended
file for local rules) to allow anyone on the system to use uinput:

```
KERNEL=="uinput", MODE="0666"
```

A more secure method uses the following line to allow anyone in <group> to use
wminput, and adds only the desired users to <group>:

KERNEL=="uinput", GROUP="<group>"

A uinput group can be created specifically for this purpose, or an existing
group such as wheel can be used.

Key symbols can be assigned to wminput button events. While standard keys are
supported by X by default, non-standard key symbols, and mapping actions to
those symbols, is not automatic. An excellent tutorial at
[HOWTO Use Multimedia Keys](http://gentoo-wiki.com/HOWTO_Use_Multimedia_Keys)
can help you set this up. An overview of the process described there:

1. Assign the key symbol in question to a button in a wminput configuration
file.
2. Use wminput and the wiimote to generate that key symbol, using xev to find
out if the key symbol is already mapped, and find the key code if it is not.
3. If the code is not mapped to the appropriate symbol, edit `~/.Xmodmap`, and
use xmodmap to map them. (A copy of my `~/.Xmodmap` is included in `cwiid/doc`)
4. Use xbindkeys or a window manager-specific utility to map the key symbols to
specific actions.

Wiimote buttons can be mapped to modifier keys (control, shift, alt, etc.) with
the command `xmodmap` command 'add [mod] = [keysym]', where keysym is the key
symbol mapped in the process described above, and mod is one of the modifier
names listed by `xmodmap -pm` (mod1 = alt).

CONFIGURATION FILES
-------------------

Configuration files are installed in `/usr/local/etc/cwiid/wminput`.
Configuration search directory order is `~/.cwiid/wminput`,
`/usr/local/etc/cwiid/wminput`.

Configuration files specify the mapping of button and key symbols to wiimote
buttons, and axis symbols to wiimote axes. The grammar is as follows (words in
angle brackets are to be replaced by appropriate values or strings, without the
brackets):

```
# <text>
```

Comment

```
include <inc_file>
```

Include the file specified by `inc_file`.

```
Wiimote.<button> = <button_symbol>
Nunchuk.<button> = <button_symbol>
Classic.<button> = <button_symbol>
Plugin.<plugin>.<button> = <button_symbol>
```

Map the button or key symbol specified on the right-hand side to the button
event specified on the left-hand side. Button and key symbols are listed in
`/usr/include/linux/input.h` (`BTN_*` and `KEY_*` macros), and in
`cwiid/wminput/action_enum.txt`. All valid wiimote, nunchuk, and classic
buttons are listed in cwiid/doc/wminput.list.

```
Wiimote.<axis> = [-][~]<abs_axis_symbol> | [-]<rel_axis_symbol>
Nunchuk.<axis> = [-][~]<abs_axis_symbol> | [-]<rel_axis_symbol>
Classic.<axis> = [-][~]<abs_axis_symbol> | [-]<rel_axis_symbol>
Plugin.<plugin>.<axis> = [-][~]<abs_axis_symbol> | [-]<rel_axis_symbol>
```

Map the axis symbol specified on the right-hand side to the axis event
specified on the left-hand side. Axis symbols are listed in
`/usr/include/linux/input.h` (`ABS_*` and `REL_*` macros), and in
`cwiid/wminput/action_enum.txt`. All valid wiimote, nunchuk, and classic axes
are listed in cwiid/doc/wminput.list. A `-` before the axis symbol inverts
the axis. A `~` is usually required before an absolute axis symbol in order
to use it for cursor movement.

```
Plugin.<plugin>.<param> = <int> | <float>
```

Set the value of a plugin parameter. Ints are silently cast to floats, and
floats are cast to ints with a warning.

```
Wiimote.Rumble = On | Off
```

Activates or deactivates force feedback/rumble function.

An event device is always created. A mouse device is created if `REL_X`, `REL_Y`,
and `BTN_LEFT` symbols are mapped (`~ABS_X` and `~ABS_Y` effectively map `REL_X` and
`REL_Y`, respectively). A joystick device is created if `ABS_X` is mapped.

PLUGINS
-------

Plugins provide a mechanism for more advanced wiimote event programming. The
following plugins are provided:

`ir_ptr` - rough IR tracking algorithm. Provides X and Y axes representing the
calculated cursor position.

`acc` - accelerometer calculations. Provides X and Y relative axes for cursor
tracking, and Roll and Pitch axes.

`nunchuk_acc` - nunchuk accelerometer calculations. Identical to acc, but using
the nunchuk accelerometers.

Plugins are by default installed in `/usr/local/lib/cwiid/plugins`. Plugin search
directory order is `~/.cwiid/plugins`, `/usr/local/lib/cwiid/plugins`.

For developers, the plugin API is specified in `cwiid/wminput/wmplugin.h`. The
examples cover most of the functionality, except buttons, which are triggered
by asserting the i^th bit of the buttons element of struct `wmplugin_data`, where
i is the index of the button.

Plugins may now be implemented in Python, as well as C. A Python version of the
acc plugin may be found in `cwiid/wminput/plugins/acc`.

