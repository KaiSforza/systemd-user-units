#systemd-user-units

##Converting to `systemd --user`

systemd is useful for system initialization, but it it also useful from a user
standpoint. Using `systemctl --system` to do anything requires root prevelages
and is therefore not useful to unprivelaged users. Starting their personal
services either requires them to be enabled or to be started by someone with
root privelages. Recently, I have begun working with the user part of systemd.
This allows me to use systemd to start unprivelaged services, such as my
window manager, tmux session, music player daemon and more.

I began using gtmanfred's guide [1] to setting this up. I am going to rewrite
it partially to be of more help, and partially to help get this into my head
even more. 

All of your systemd user units will go to $HOME/.config/systemd/user. These
units take precedence over units in other systemd unit directories.

### Prerequisites

There are two packages you need to get this working, both currently available
from the AUR: xorg-launch-helper [2] and user-session-units [3]. You should also
have at least the ability to apply patches and edit pkgbuilds. Just remember
that this is no easy feat. Though once you get it working you will be rewarded
with a damn beautiful system.

### Creating some starting units

Next is setting up your targets. I set up two, one for my window manager (i3)
and another as a default target. The window manager target, `wm.target`, and 
populated it like so:

    [Unit]
    Description=Window manager target
    Wants=xorg.target
    Wants=myStuff.target
    Requires=dbus.socket
    AllowIsolate=true
    
    [Install]
    Alias=default.target

This will be the target for your graphical interface.

I put together a second target called `mystuff.target`. This will be 'WantedBy'
all services but your window manager:

    [Unit]
    Description=Xinitrc Stuff
    Wants=wm.target
    
    [Install]
    Alias=default.target

Link this unit to default.target. When you start systemd --user, it will start
this target. 

Next you need to begin writing services. First you should throw together a
service for your window manager. I named mine `i3.service`:

    [Unit]
    Description=Simple tiling/stacking/tabbed window manager
    Before=mystuff.target
    After=xorg.target
    
    [Service]
    Requires=xorg.target
    Environment=PATH=/home/wgiokas/.cabal/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/home/wgiokas/bin
    Environment=DISPLAY=:0
    ExecStart=/usr/bin/i3
    Restart=always
    RestartSec=10

    [Install]
    WantedBy=wm.target

Note the [Install] section includes a 'WantedBy' part. When using `systemctl
--user enable` it will link this as
$HOME/.config/systemd/user/wm.target.wants/i3.service, allowing it to be
started at login. I would recommend enabling this service, not linking it
manually.

You can fill your user unit directory with a plethora of services, I currently
have ones for mpd, gpg-agent, offlineimap, parcellite, pulse, tmux, urxvtd,
xbindkeys and xmodmap to name a few.

### Some actual important stuff

Doing this will not allow you to run systemd --user, though. If you installed
user-session-units as listed above, then you must copy `user-session@.service`
to your user unit directory and use `systemctl --user enable
user-session@yourloginname.service`. Next, edit the `user-session@.service` (not
the `user-session@yourloginname.service`) and replace this line: 

    Environment=DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/%I/dbus/user_bus_socket

with this:

    Environment=DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/%U/dbus/user_bus_socket

You will need to patch systemd to do this, so either using systemd-git from
after commit 067d851d [4] or patch it into systemd with the ABS. 

Last but not least, add this line to /etc/pam.d/login or
/etc/pam.d/systemd-auth (whichever exists):

    session    required    pam_systemd.so

Now add `/usr/lib/systemd/systemd --user` to your shell's `$HOME/.*profile` file
and you are ready to go! (This takes a lot of tweaking, so when I say that, I
mean that you are ready to debug and find spelling mistakes.)

### Anecdotes

One of the most important things you can add to the service files you will be
writing is the use of Before= and After= in the [Unit] section. These two
parts will determine the order things are started. Say you have a graphical
application you want to start on boot, you would put `After=xorg.target` into
your unit. Say you start ncmpcpp, which requires mpd to start, you can put
`After=mpd.service` into your ncmpcpp unit. You will eventually figure out
exactly how this needs to go either from experience or from reading the
systemd manual pages. I would recommend starting with systemd.unit(5) [5].

Just as a note to readers and users, I am by no means an expert on this stuff.
It's not easy, and it made me work. If you don't get it to work the first time,
don't just pass it off as a failed attempt. I completely botched it my first
try, and only got it working with some more reading and excellent help. Don't
give up!

### Links

* [1] http://blog.gtmanfred.com/?p=26
* [2] https://aur.archlinux.org/packages/xorg-launch-helper/
* [3] https://aur.archlinux.org/packages/user-session-units/
* [4] http://cgit.freedesktop.org/systemd/systemd/commit/?id=067d851d30386c553e3a84f59d81d003ff638b91
* [5] http://www.freedesktop.org/software/systemd/man/systemd.unit.html
