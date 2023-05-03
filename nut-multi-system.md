# NUT Installation and Configuration

## NUT Master

Update packages and install `nut` package.

```bash
cd
sudo apt-get update
sudo apt-get install nut
```

Edit `/etc/nut/ups.conf` to add information about the UPS connected to the system.

```bash
sudo vi /etc/nut/ups.conf
```

Copy-and-paste the following into the `ups.conf` file.

- Replace `<UPS_NAME>` with a short, descriptive name for the UPS. (I believe spaces are not allowed.)

- Replace `<UPS DESCRIPTION>` with a longer description of the UPS. (Spaces are allowed.)
- Replace the value of `override.battery.runtime.low` to number of minutes of battery life left on the UPS when the NUT server will instruct all NUT clients to shut down.
- If `usbhid-ups` does not automatically find your USB UPS, you may need a custom configuration here. See docs.

```
[<UPS_NAME>]
        driver = usbhid-ups
        port = auto
        desc = "<UPS DESCRIPTION>"
        ignorelb
        override.battery.runtime.low = 15
```

Save and close the file.

Edit the upsd.users file to add a username and password for NUT client authentication.

```bash
sudo vi /etc/nut/upsd.conf
```

Add the following section to the `upsd.users` file:

- `upsmon` is the username here. You can change it, but you'd have to change it elsewhere too.
- Change `<P@ssw0rd!` to the password you want to use for the `upsmon` account.

```
[upsmon]
        password  = <P@ssw0rd!>
        upsmon master
```

Save and close the file.

Edit the `/etc/nut/nut.conf` file to set which type of NUT configuration we are setting up on this system.

```bash
sudo vi /etc/nut/nut.conf
```

Set `netserver` mode for the primary (master) NUT instance.

```
MODE = netserver
```

Save and close the file.

```bash
sudo vi /etc/nut/upsd.conf
```

Add `LISTEN` lines for each host that is allowed to talk to the NUT server. `0.0.0.0` should work for all, but you may want to specify specific IP addresses.

- Change <PORT> to the port number you want the NUT server to listen on. I am not aware of any default values for this.

```
LISTEN 0.0.0.0 <PORT>
```

Save and close the file.

Edit the /etc/nut/upsmon.conf file to tell it which UPS to monitor.

```bash
sudo vi /etc/nut/upsmon.conf
```

Add a MONITOR line:

- Change <UPS_NAME> to the same value set in ups.conf above.

- Change <IP ADDRESS> to the IP address of the NUT server. For the master this should likely be 127.0.0.1.

- Change <PORT> to the port set in the upsd.conf file above.

- Change <P@ssw0rd!> to the password set in the 

```
MONITOR <UPS_NAME>@<IP ADDRESS>:<PORT> 1 upsmon <P@ssw0rd!> master
```

Save and close the file.

Enable and start the following:

- nut-driver - The driver that speaks directly to the UPS

- nut-monitor - Monitors the UPS and decides when to take action.

- nut-server - Communicates with NUT clients about UPS status.

```bash
sudo systemctl enable --now nut-driver nut-server nut-monitor
```

See if the UPS is online:

- Change <UPS_NAME> to the same value set in ups.conf above.

- Change <IP ADDRESS> to the IP address of the NUT server. For the master this should likely be 127.0.0.1.

- Change <PORT> to the port set in the upsd.conf file above.

```bash
upsc <UPS_NAME>@<IP ADDRESS>:<PORT>
```

Your output should look similar to the following:

```
Init SSL without certificate database
battery.charge: 74
battery.runtime: 1167
battery.runtime.low: 15
battery.type: PbAC
battery.voltage: 26.3
battery.voltage.nominal: 24.0
device.mfr: Tripp Lite
device.model: Tripp Lite UPS
device.type: ups
driver.flag.ignorelb: enabled
driver.name: usbhid-ups
driver.parameter.pollfreq: 30
driver.parameter.pollinterval: 2
driver.parameter.port: auto
driver.parameter.synchronous: no
driver.version: 2.7.4
driver.version.data: TrippLite HID 0.82
driver.version.internal: 0.41
input.frequency: 60.1
input.voltage: 116.4
output.frequency.nominal: 60
output.voltage: 116.4
output.voltage.nominal: 120
ups.beeper.status: enabled
ups.delay.shutdown: 20
ups.load: 20
ups.mfr: Tripp Lite
ups.model: Tripp Lite UPS
ups.power: 0.0
ups.power.nominal: 1500
ups.productid: 2012
ups.status: OL CHRG
ups.timer.reboot: 65535
ups.timer.shutdown: 65535
ups.vendorid: 09ae
ups.watchdog.status: 0
```

## NUT Client

Update packages and install nut-client package.

```bash
cd
sudo apt-get update
sudo apt-get install nut-client
```

Edit the /etc/nut/nut.conf file to set which type of NUT configuration we are setting up on this system.

```bash
sudo vi /etc/nut/nut.conf
```

Set netclient mode for the secondary (slave) NUT instance.

```
MODE = netclient
```

Save and close the file.

Edit the /etc/nut/upsmon.conf file to tell it which UPS to monitor.

```bash
sudo vi /etc/nut/upsmon.conf
```

Add a MONITOR line:

- Change <UPS_NAME> to the same value set in ups.conf in the server section above.

- Change <IP ADDRESS> to the IP address of the NUT server. For the slave this is likely a non-localhost IP address.

- Change <PORT> to the port set in the upsd.conf file in the server section above.

- Change <P@ssw0rd!> to the password set in the uspd.users file in the server section above.

```
MONITOR <UPS_NAME>@<IP ADDRESS>:<PORT> 1 upsmon <P@ssw0rd!> slave
```

Save and close the file.

Enable and start the following:

- nut-client - Monitors the server instance for UPS information. FYI: Under the hood this is just running nut-monitor so systemctl status might not show the correct name.

```bash
sudo systemctl enable --now nut-client
```

See if the UPS is online:

- Change <UPS_NAME> to the same value set in ups.conf in the server section above.

- Change <IP ADDRESS> to the IP address of the NUT server. For the secondary instance this should be a non-localhost address.

- Change <PORT> to the port set in the upsd.conf file in the server section above.

```bash
upsc <UPS_NAME>@<IP ADDRESS>:<PORT>
```

Your output should look similar to the following:

```
Init SSL without certificate database
battery.charge: 74
battery.runtime: 1167
battery.runtime.low: 15
battery.type: PbAC
battery.voltage: 26.3
battery.voltage.nominal: 24.0
device.mfr: Tripp Lite
device.model: Tripp Lite UPS
device.type: ups
driver.flag.ignorelb: enabled
driver.name: usbhid-ups
driver.parameter.pollfreq: 30
driver.parameter.pollinterval: 2
driver.parameter.port: auto
driver.parameter.synchronous: no
driver.version: 2.7.4
driver.version.data: TrippLite HID 0.82
driver.version.internal: 0.41
input.frequency: 60.1
input.voltage: 116.4
output.frequency.nominal: 60
output.voltage: 116.4
output.voltage.nominal: 120
ups.beeper.status: enabled
ups.delay.shutdown: 20
ups.load: 20
ups.mfr: Tripp Lite
ups.model: Tripp Lite UPS
ups.power: 0.0
ups.power.nominal: 1500
ups.productid: 2012
ups.status: OL CHRG
ups.timer.reboot: 65535
ups.timer.shutdown: 65535
ups.vendorid: 09ae
ups.watchdog.status: 0
```

