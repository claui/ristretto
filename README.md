# Ristretto

Ristretto is a service which, when triggered, gives your Mac a quick caffeine shot to keep it awake for 30 seconds.

It is implemented as a launchd wrapper for the built-in `caffeinate` tool. This makes it independent from the calling process.

## What problem does Ristretto solve?

If your Mac serves SSH but has the `ttyskeepawake` assertion for power management, you can call the Ristretto service whenever you need your Mac to stay awake during short gaps, e. g. between SSH sessions.


# Installation

1. Make sure you have [Homebrew](https://brew.sh) installed.

2. Run `brew tap claui/public` if you haven’t already done so.

3. Run:

```
brew install ristretto
```


# Usage

## Enabling the service

To run the service, you need to enable it first. It will then stay enabled, even through subsequent restarts, until you disable the service.

The following command line enables the service:

```
launchctl bootstrap "gui/$(id -u)" \
  /Library/LaunchAgents/homebrew.mxcl.ristretto.plist
```

## Checking the service status

The following command line checks whether the service is currently enabled or running:

```
launchctl print "gui/$(id -u)" | grep -F homebrew.mxcl.ristretto
```

If `launchctl` prints no result, the service is not enabled. If the output starts with the number `0`, the service is currently enabled but not running. Otherwise, the service is enabled and running.

## Running the service

```
launchctl kickstart -k "gui/$(id -u)"/homebrew.mxcl.ristretto
```

## Disabling the service

The following command line disables the service:

```
launchctl bootout "gui/$(id -u)"/homebrew.mxcl.ristretto
```


# FAQ

## Why do I need a 30-second gap between SSH sessions?

**tl;dr** You probably won’t but I do.

I wrote Ristretto to fulfill a specific need I have with my local network backup. The background is that I have several devices at home which run the ZFS filesystem. Those systems are configured to create hourly snapshots using the `znapzend` service.

My backup strategy for those devices is to keep those snapshots separate from the machines themselves, similar to how a Time Capsule makes backups from an APFS or HFS+ filesystem.

To recreate the Time Capsule experience for ZFS, I run an old Mac mini in my local network, whose only role is to `zfs receive` snapshots from other machines via SSH. Just like snapshot creation, `znapzend` provides such a feature as well. However, I ran into an issue when I tried to configure `znapzend` to run a few post-send housekeeping tasks: `znapzend` uses separate SSH sessions for `zfs send` and the post-send tasks. Therefore, I want my Mac mini to accept a series of one-line commands, each wrapped into a separate SSH session.

However, I use a mechanical hard disk for my backups; therefore, I want to keep noise at manageable levels. To achieve this, I need my Mac mini to sleep by default, and I only want it to wake up while `znapzend` is talking to it. To achieve this, I use macOS’s `pmset womp` setting (which allows the Mac mini to wake up when it receives an Ethernet magic packet) together with the `pmset ttyskeepawake` feature (which prevents idle system sleep when any tty is active).

As it turns out, the `ttyskeepawake` setting is quite strict; whenever an SSH session ends, the Mac will go to sleep immediately, and not wake up until the next SSH session – even if the latter is solicited mere seconds later!

This causes the following issue:

- `znapzend` asks the Mac for a SSH session to do `zfs receive`.
- The Mac receives the magic packet, wakes up and spins up the hard disks _(uuuwwwweeEEEEEE.)_
- When the SSH session ends, the `ttyskeepawake` power management assertion goes away immediately.
- The Mac goes to sleep and spins down the hard disks _(EEEEEEEeeuuuuuwwwwww.)_
- Seconds later, `znapzend` asks the Mac for another SSH session.
- The Mac receives the magic packet, wakes up and spins up the hard disks _(uuuwwwweeEEEEEE)_ …
- … and so on.

I could not find any immediate command-line solution for this. I tried several things to give my Mac a quick caffeine shot of a few seconds, hoping that it would make it stay awake during the gap between SSH sessions. For example, I tried to use `caffeinate` to set additional `pmset` assertions, and tried to put them in the background using `disown`, `screen`, or `nohup`; ultimately, none of this worked because whenever the SSH session ended, the `caffeinate` process received a `HUP` signal and exited.

The solution that ultimately worked was to provide a service which runs `caffeinate`, allows any user to manually start it on demand, and survives SSH disconnects and `HUP` signals.


# License

Copyright (c) 2018 Claudia <clau@tiqua.de>

Permission to use, copy, modify, and/or distribute this software for
any purpose with or without fee is hereby granted, provided that the
above copyright notice and this permission notice appear in all
copies.

THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL
WARRANTIES WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED
WARRANTIES OF MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE
AUTHOR BE LIABLE FOR ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL
DAMAGES OR ANY DAMAGES WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR
PROFITS, WHETHER IN AN ACTION OF CONTRACT, NEGLIGENCE OR OTHER
TORTIOUS ACTION, ARISING OUT OF OR IN CONNECTION WITH THE USE OR
PERFORMANCE OF THIS SOFTWARE.
