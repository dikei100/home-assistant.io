---
title: Wake on LAN
description: Instructions on how to setup the Wake on LAN integration in Home Assistant.
ha_category:
  - Network
  - Switch
ha_release: 0.49
ha_iot_class: Local Push
ha_domain: wake_on_lan
ha_platforms:
  - switch
ha_codeowners:
  - '@ntilley905'
ha_integration_type: integration
---

The **Wake on LAN** {% term integration %} enables the ability to send _magic packets_ to [Wake on LAN](https://en.wikipedia.org/wiki/Wake-on-LAN) capable devices to turn them on.

There is currently support for the following device types within Home Assistant:

- [Switch](#switch)

## Configuration

To use this {% term integration %} in your installation, add the following to your `configuration.yaml` file:

```yaml
# Example configuration.yaml entry
wake_on_lan:
```

### Integration services

Available services: `send_magic_packet`.

#### Service `wake_on_lan.send_magic_packet`

Send a _magic packet_ to wake up a device with 'Wake on LAN' capabilities.

| Service data attribute    | Optional | Description                                             |
|---------------------------|----------|---------------------------------------------------------|
| `mac`                     |       no | MAC address of the device to wake up.                   |
| `broadcast_address`       |      yes | Optional broadcast IP where to send the magic packet.   |
| `broadcast_port`          |      yes | Optional port where to send the magic packet.           |

Sample service data:

```json
{
   "mac":"00:40:13:ed:f1:32"
}
```

<div class='note'>
This usually only works if the target device is connected to the same network. Routing the magic packet to a different subnet requires a special configuration on your router or may not be possible.
The service to route the packet is most likely named "IP Helper". It may support Wake on LAN, but not all routers support this.
</div>

## Switch

The `wake_on_lan` (WOL) switch {% term integration %} allows you to turn on a [WOL](https://en.wikipedia.org/wiki/Wake-on-LAN) enabled computer.

### Switch configuration

The WOL switch can only turn on your computer and monitor the state. There is no universal way to turn off a computer remotely. The `turn_off` variable is there to help you call a script when you have figured out how to remotely turn off your computer. See below for suggestions on how to do this.

It's required that the binary `ping` is in your `$PATH`.

To enable this switch in your installation, add the following to your `configuration.yaml` file:

```yaml
# Example configuration.yaml entry
switch:
  - platform: wake_on_lan
    mac: MAC_ADDRESS
```

{% configuration %}
mac:
  description: "The MAC address to send the wake up command to, e.g, `00:01:02:03:04:05`."
  required: true
  type: string
name:
  description: The name of the switch.
  required: false
  default: Wake on LAN
  type: string
host:
  description: The IP address or hostname to check the state of the device (on/off). If this is not provided, the state of the switch will be assumed based on the last action that was taken.
  required: false
  type: string
turn_off:
  description: Defines an [action](/getting-started/automation/) to run when the switch is turned off.
  required: false
  type: string
broadcast_address:
  description: The IP address of the host to send the magic packet to.
  required: false
  default: 255.255.255.255
  type: string
broadcast_port:
  description: The port to send the magic packet to.
  required: false
  type: integer
{% endconfiguration %}

### Examples

Here are some real-life examples of how to use the **turn_off** variable.

#### Suspending Linux

Suggested recipe for letting the `turn_off` script suspend a Linux computer (the **target**)
from Home Assistant running on another Linux computer (the **server**).

1. On the **server/Homeassistant**, log in as the user account Home Assistant is running under. In this example it's `hass`. You can also use an addon like "Advanced SSH & Web Terminal" to access the terminal.
2. On the **server/Homeassistant**, create SSH keys by running `ssh-keygen`. Just press enter on all questions. You do not need to create a password for the keys.
3. On the **server/Homeassistant**, copy the newly generated keys from /root/.ssh/ to /config/.ssh, for example by typing in the terminal 'cp /root/.ssh/id_edxxxxx /config/.ssh/' and 'cp /root/.ssh/id_edxxxxx.pub /config/.ssh/' (replace id_edxxxxx with the name of the newly generated keys!).
4. (optional) On the **target/your computer**, create a new account that Home Assistant can ssh into: `sudo adduser hass`. Just press enter on all questions except password. It's recommended using the same user name as on the server. If you do, you can leave out `hass@` in the SSH commands below.
5. On the **server/Homeassistant**, transfer your public SSH key by `ssh-copy-id USERNAME@TARGET` where USERNAME is the username on your target/computer and TARGET is your target/computers machine's name or IP address. You can either use a user that already exists on the target/computer or the one you just created in step 4. Enter the users password or, if you created a new user, the password you created in step 4.
6. On the **server/Homeassistant**, verify that you can reach your target machine without password by `ssh USER@TARGET-IP-Adress`, for example 'ssh hass@192.168.156.123'. To close the ssh connection, type 'exit' and press enter.
7. On the **target/your computer**, we need to let the user execute the program needed to suspend/shut down the target computer. Here it is `systemctl`.
8. On the **target/your computer**, using an account with sudo access (typically your main account), `sudo visudo`. Add this line to the bottom of the file: `USER ALL=NOPASSWD:/usr/sbin/systemctl`.
9. On the **server/Homeassistant**, add the following to your configuration, replacing TARGET with the target's name:

```yaml
switch:
  - platform: wake_on_lan
    name: "TARGET"
    ...
    turn_off:
      service: shell_command.turn_off_TARGET

shell_command:
  turn_off_TARGET: "ssh -i /config/.ssh/id_edxxxxx -o 'StrictHostKeyChecking=no' USER@TARGETS-IP-ADRESS sudo systemctl suspend" # (dont forget to change USER to the username of your targets machine and TARGETS-IP-ADRESS to the IP-adress of the target machine! Also change id_edxxxxx to the name of your keys-file, without the .pub!)
```

If you wish, you can replace 'suspend' with other systemctl commands.

## Helper button with automation

A switch defined with the `wake_on_lan` platform will render in the UI with both 'on' and 'off' clickable actions. If you don't intend to use the `turn_off` functionality, then using a virtual button & automation will look cleaner and less confusing. It will only have one action.

1. First, define a new helper button. 
    - Go to **{% my helpers title="Settings > Devices & Services > Helpers" %}** and select the **+ Create helper** button. Choose **Button** and give it a name. A button named "Wake PC" will render like this:

    ![image](https://github.com/home-assistant/home-assistant.io/assets/252209/10e468a0-45c8-4ee7-b69d-596db3845b14)

2. Then, create a new automation. Go to **{% my automations title="Settings > Automations & scenes" %}** and select **+ Create Automation**. 
    - The trigger will be on `State` and the entity will be the button you created. 
    - Continuing your example, the trigger YAML will look like this:

      ```yaml
      platform: state
      entity_id:
        - input_button.wake_pc
      ```

3. For the action, select **Call service** and choose **Wake on LAN: Send magic packet**.
4. Type in the target MAC address.
    - Do not change the broadcast port unless you've configured your device to listen to a different port.
    - Continuing our example, the action YAML looks like this:

      ```yaml
      service: wake_on_lan.send_magic_packet
      data:
        broadcast_port: 9
        mac: 00:11:22:33:44:55
      ```

5. Save the automation. Now, when you activate `PRESS` on the helper button in the UI, Home Assistant will send a wake packet to the configured MAC.
