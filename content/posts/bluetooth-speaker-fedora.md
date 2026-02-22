+++
title = 'Using Fedora as a Bluetooth Speaker'
date = 2026-02-21T19:20:00-04:00
+++

I had a pair of PC speakers sitting in my drawer.
I also had a mini-PC not being used.
I figured that I could put the two together to make my own Bluetooth speaker.
This post describes the steps I took to make that happen.

## OS Installation

I chose Fedora because it is the Linux distribution I'm most familiar with.
I first tried to do this with the [Fedora IoT](https://fedoraproject.org/iot/) spin.
However, I ran into various problems along the way due to that being an extremely stripped down version of the OS, and to its immutable filesystem nature.
I wanted to make faster progress so I abandoned that idea in favor of the Fedora Server spin.

I used the standard Anaconda installation by writing the Fedora Server 43 ISO to an USB drive.

I left the root account disabled, and created an admin user called `speaker` that also required a password.

This device was planned to connect to my home network via Wi-Fi only. 
In fact, no wired network was available during the installation either.
By connecting to my Wi-Fi from the installer, the Wi-Fi password persists on the device.
This was a nice surprise.

The full disk was used for the installation.
I did not configure disk encryption because that would require a password during boot which wasn't an option for me.

I set the host name to `speaker` because the host name is the default value used for the Bluetooth device.

Once the installation finished, I rebooted the system and started configuring the system.

## Sound

Fedora Server doesn't come with the required audio packages which makes sense given that it's a server edition.
I installed the following packages:

```bash
sudo dnf install alsa-utils wireplumber
```

That installs a bunch of dependent packages as well.

Next, I started the user audio services and made them start automatically going forward:

```bash
systemctl --user enable --now pipewire pipewire-pulse wireplumber
systemctl --user status pipewire pipewire-pulse wireplumber
```

## Bluetooth

The Bluetooth packages were already installed.
Here is how I verified the Bluetooth services were running:

```bash
sudo systemctl status bluetooth
bluetoothctl show
```

By default, Bluetooth is configured to connect to other devices.
I wanted the opposite.
I wanted another device, e.g. my phone, to connect to this system:

```bash
mkdir -p ~/.config/wireplumber/wireplumber.conf.d/
cat <<EOF > ~/.config/wireplumber/wireplumber.conf.d/10-bluetooth-sink.conf
monitor.bluez.properties = {
  bluez5.roles = [ a2dp_sink a2dp_source ]
}
EOF

# Restart wireplumber for the changes to take effect
systemctl --user restart wireplumber
```

## Headless

So far, things should be mostly working.
However, there are two problems.

First, the user services don't start on boot.
They only start after a user logs in.
Linger is how we tell the system to run the user services regardless:

```bash
sudo loginctl enable-linger $USER
```

Second, even with linger, you'll notice that the audio devices don't quite work without a user logged in.
Frustratingly, it requires a user logged in locally (login via ssh doesn't count!).
I learned about [seat monitoring](https://pipewire.pages.freedesktop.org/wireplumber/daemon/configuration/bluetooth.html#logind-integration) which implements that restriction.
Here's how to disable it for this use case:

```bash
cat <<EOF > ~/.config/wireplumber/wireplumber.conf.d/disable-seat-monitoring.conf
wireplumber.profiles = {
  main = {
    monitor.bluez.seat-monitoring = disabled
  }
}
EOF

# Restart wireplumber to apply the changes
systemctl --user restart wireplumber
```

## Connecting

The next step is to connect the device, e.g. phone, to the system.

Enter the Bluetooth tool shell by running `bluetoothctl`. 
Inside this shell, run the following:

```text
power on
agent on
default-agent
discoverable on
pairable on
```

Then, from another device, the system is now visible.
In this case, `speaker` shows up in the list of available devices on my phone.
Initiate the Bluetooth pairing from the device.
A number will show up on the devices and on the `bluetoothctl` shell on the system.
Compare them and accept on both.
Do this somewhat fast as there is a timeout for this operation.
Once the connection is established, mark the device as trusted so it is easier to connect later on.
From the `bluetoothctl` shell:

```text
trust <mac>
```

Replace `<mac>` with the connecting device's MAC address. 
This should be obvious from the text that has just scrolled through the screen.
If you can't find it, type `devices`.
This will list the device that has just connected.
You can use tab-completion to facilitate entering the MAC address.
Exit the `bluetoothctl` shell by typing `exit`.

## Enjoy the Music

Cool!
Now I have my own Bluetooth speaker made up of spare parts.

At some point, I want to give Fedora IoT another shot.
There was a lot of trial and error to get all of this working.
At least now, I know how to do that on Fedora Server.
Making that happen on Fedora IoT should be an easier task now.

This was certainly a learning experience.
I definitely value how the Bluetooth feature I needed was already there.
I just had to figure out how to use it.
I hope this post is useful to someone also trying this. 

## Bonus

To change the volume from the CLI, use the `wpctl` tool:

```bash
wpctl set-volume <ID> 0.95
```

That sets the volume to 95%. 
Replace <ID> with the audio device. 
You can get it from the output of `wpctl status`.
