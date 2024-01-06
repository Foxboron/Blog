---
title: "Stream to chromecast with resolved, vlc and bash"
date: "2024-01-06T20:59:20+02:00"
tags:
- linux
- resolved
---

Chromecast is one of those devices I just generally use a lot. They are small
practical and enables me to stream video or music to my TV from multiple
devices. But it also requires you to have a supported browser or video player.
This is obviously a bit boring.

There has been multiple command line chromecast streamers through the years. But
their ffmpeg usage has been shoddy at best with no hardware decoding support and
usually quite bad implementations.

Instead of using these tools I have wound up with two bash scripts that pipes
together a command line program to stream towards chromecasts and it works
surprisingly well.


The first part of this is how we stream towards the chromecast. This is easily
solveable with `vlc` luckily. You can just open `vlc file://....`, select
`Playback -> Renderer` and generally find the correct chromecast there. But this
is boring. We have to manually open the GUI and select the device!

I can peak into the app on my phone and learn that the local IP of my chromecast
is `192.168.1.231`. With `clvc` you can also pass `sout` and
`sout-chromecast-ip` to auto-start a headless VLC players that will connect
towards your chromecast.

{{< highlight bash >}}
IP="192.168.1.231"
cvlc --sout "#chromecast" --sout-chromecast-ip="$IP" "file://...."
{{< /highlight >}}

This works really well on it's own. You can also pipe `yt-dlp` into `clvc` and
support most media websites like youtube or twitch. This allows you to quickly
write up some scripts to stream from these sites and is fairly flexible.

{{< highlight  bash >}}
IP="192.168.1.231"
yt-dlp "https://twitch.tv/sovietwomble" -o - | \
cvlc --sout "#chromecast" --sout-chromecast-ip="$IP" -
{{< /highlight >}}

There is one issue though, that IP is not static. If the chromecast reconnects
to the network it might change it's IP, and I'm never going to bother setting a
static device.

Luckily chromecast announces itself over [DNS Service Discovery](http://www.dns-sd.org/) 
which allows devices to announce themself over the network through DNS names. To
query for these devices we will be using `systemd-resolved` and `resolvectl`.
You can probably use another DNS query tool, but `systemd-resolved` also
supports `mDNS` which is useful for finding services on the network.

First you need to enable `systemd-resolved` through `systemctl enable --now
systemd-resolved`. Your mileage might wary but in some distributions this comes
enabled by default. It's also important to note that you don't need to make
`resolved` your default stub resolver for any of this to work.

If you do read the [`resolvctl(1)` man page](https://man.archlinux.org/man/resolvectl.1#COMMANDS)
you will see it has native support for dns-sd through `resolvectl service`. 

{{< highlight  bash >}}
$ resolvectl service _googlecast._tcp local
{{< /highlight >}}

However I have not gotten this to work as it seems like it will try to send a
combined DNS query to your network DNS resolver, and this doesn't seem supported
depending on the router you have. So we have to implement this ourself. So we
will be using the `PTR+SRV+TXT` process as outlined in [RFC6763](https://www.ietf.org/rfc/rfc6763.txt).

To find available chromecasts on the network we'll first query for PTR records
on `_googlecast._tcp.local`.

{{< highlight bash >}}
$ resolvectl --type=PTR query _googlecast._tcp.local
_googlecast._tcp.local IN PTR Chromecast-Ultra-13ccb56dd0ea6ce65dbe15b39c573856._googlecast._tcp.local
{{< /highlight >}}


Here I see one Chromecast. Then we need to query for the service record it self,
SRV is defined as part of [RFC2782](https://tools.ietf.org/html/rfc2782).

{{< highlight bash >}}
$ resolvectl --type=SRV query Chromecast-Ultra-13ccb56dd0ea6ce65dbe15b39c573856._googlecast._tcp.local
Chromecast-Ultra-13ccb56dd0ea6ce65dbe15b39c573856._googlecast._tcp.local IN SRV 0 0 8009 13ccb56d-d0ea-6ce6-5dbe-15b39c573856.local
{{< /highlight >}}

And finally we can just query the TXT record of the service for the IP of the
device itself, which is the correct IP we started out with :)!

{{< highlight bash >}}
$ resolvectl query 13ccb56d-d0ea-6ce6-5dbe-15b39c573856.local
192.168.1.231
{{< /highlight >}}

To make this easier in the future we can easily parse this space limited output
with `read` in a bash script[^1].

{{< highlight bash >}}
$ ~/.local/bin/dns-sd
#!/usr/bin/bash
set -e
read -r _ _ _ ptr _ < <(resolvectl --type=PTR query "$1")
read -r _ _ _ _ _ _ srv _ < <(resolvectl --type=SRV query "$ptr")
read -r _ ip _ < <(resolvectl query "$srv")
printf "$ip"
{{< /highlight >}}

And we can make our `chromecast` script as well which calls `dns-sd`.

{{< highlight bash >}}
$ ~/.local/bin/chromecast
#!/usr/bin/bash
IP="$(dns-sd _googlecast._tcp.local)"
echo "Using $IP"
yt-dlp "$1" -o - | cvlc --sout "#chromecast" --sout-chromecast-ip="$IP" -
{{< /highlight >}}


With this we have a neat way to stream web content to our chromecast without
having to go through a GUI.

### Bonus content: qutebrowser

If you, like me, enjoy qutebrowser but find the lack of chromecast support
annoying we can resolve that with the script above.

We'll use the current chromecast script from qutebrowser:

https://github.com/qutebrowser/qutebrowser/blob/main/misc/userscripts/cast

And apply the diff here, which just removes code we don't really need.

{{< highlight diff >}}
diff --git a/misc/userscripts/cast b/misc/userscripts/cast
index ec703d5fb..57c8f95f9 100755
--- a/misc/userscripts/cast
+++ b/misc/userscripts/cast
@@ -139,37 +139,6 @@ printjs() {
 }
 echo "jseval -q $(printjs)" >> "$QUTE_FIFO"

-tmpdir=$(mktemp -d)
-file_to_cast=${tmpdir}/qutecast
-cast_program=$(command -v castnow)
-
-# pick a ytdl program
-for p in "$QUTE_CAST_YTDL_PROGRAM" yt-dlp youtube-dl; do
-    ytdl_program=$(command -v -- "$p")
-    [ "$ytdl_program" == "" ] || break
-done
-
-if [[ "${cast_program}" == "" ]]; then
-    msg error "castnow can't be found"
-    exit 1
-fi
-if [[ "${ytdl_program}" == "" ]]; then
-    msg error "youtube-dl or a drop-in replacement can't be found in PATH, and no installed program " \
-        "specified in QUTE_CAST_YTDL_PROGRAM (currently \"$QUTE_CAST_YTDL_PROGRAM\")"
-    exit 1
-fi
-
-# kill any running instance of castnow
-pkill -f -- "${cast_program}"
-
-# start youtube download in stream mode (-o -) into temporary file
-"${ytdl_program}" -qo - "$1" > "${file_to_cast}" &
-ytdl_pid=$!
-
+pkill -f -- chromecast
 msg info "Casting $1" >> "$QUTE_FIFO"
-# start castnow in stream mode to cast on ChromeCast
-tail -F "${file_to_cast}" | ${cast_program} -
-
-# cleanup remaining background process and file on disk
-kill ${ytdl_pid}
-rm -rf "${tmpdir}"
+chromecast "$1"
{{< /highlight >}}


Then add a keybind into `config.py`.

{{< highlight python >}}
config.bind(',c', 'spawn --userscript cast')
{{< /highlight >}}


Ta-da, chromecast support with our script.


[^1]: I might have written all of this to show off the script. I just really like it. #NoShame
