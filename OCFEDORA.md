# OCFEDORA 36

## Before Installation

It is recommended to install [CrashPlan](https://github.ibm.com/openclient/ocfedora/wiki/CrashPlan) to back up your home directory and other useful files on your system. Double-check important hidden directories like .ssh are included in your backup set.

It is also recommended to take a backup of your Notes ID file where is it is easily accessible or add your cell phone number to your w3 profile to allow for authentication of w3id based sites, like this page after the installation is complete.

## Downloading Fedora

**We are entitled to use ONLY the CIO approved version of Fedora**. These can be found via a link on the [installation page](https://github.ibm.com/openclient/ocfedora/wiki/Installation).

Then go [download a copy](https://getfedora.org/) of a Fedora installation image.

Other [flavors](https://archive.fedoraproject.org/pub/fedora/linux/releases/36/Spins/x86_64/iso/).

## Installing Fedora

Follow the Fedora installation guide for the version of Fedora you're installing that can be found on the [Fedora Docs](https://docs.fedoraproject.org/en-US/docs/) web site.

**NOTE:** You will need to **enable disk encryption** at install time in order to be compliant with the IBM security requirements.

## Scripted Installation of the IBM Base Layer

### Pre-Requisites:

- An approved version of Fedora Linux OS (approved versions can be found via the link on the [installation page](https://github.ibm.com/openclient/ocfedora/wiki/Installation))
- An Encrypted root filesystem.
- Recommended BIOS settings for EFI:
  - `Kernel DMA Protection = Disabled` (under Virtualization - T490, P52, X1Carbon 7th Gen, Yoga Gen 4; 2019 Models or newer)
  - `Secure Boot = Disabled` (Under Security)
  - `Boot Mode = UEFI` or `Boot Mode = Both`, if Both, then Select UEFI First.
  - `CSM = Enabled` (Compatibility Support Mode)
  - If you have an Nvidia GPU, set the `Display Mode = Discrete` (Under Config)
  - You can use Nvidia Optimus, Advanced or Hybrid Display mode settings in the BIOS, but please review Bumblebee or Prime documentation for details.
- Latest packages - Prep the system for the Linux@IBM Fedora Layer install by running (if you forget this step, the install script will check for new packages anyway, install and prompt you to reboot if necessary - just run this script again after rebooting.)
  - `sudo dnf upgrade --refresh`

### Perform the IBM Layer installation:

1. Download the [installation bundle ](https://ibm.box.com/v/fedora-installer)from box (note: this can be done without a VPN connection)

2. Run the bundle as root

   ```
   $ sudo bash ./li-fc-<CurrentVersion>.run
   ```


## Registration

Every user must register their system for use within IBM.

1. Start by [installing](https://github.ibm.com/openclient/ocfedora/wiki/Installation).

2. If your system is using UEFI Secure Boot, you need to enable CSM (Compability Support Mode) in the laptop BIOS to allow the serial number to be detected.

   - Enter bios: secure boot - disable, csm - enable

3. Run the registration tool and fill in the required information.

   ```
   python3 /opt/ibm/registration/registration.py
   ```

   - The registration tool will attempt to create a VPN connection if you're not connected to the IBM network. If you need, for some reason, to obtain a VPN connection (not necessary if you're connected to the IBM network). Note that the VPN connection you set up here is configured such that only the registration tooling is allowed, no other access to the IBM network is configured or expected to work.

     ```
     echo '_YOUR_W3ID_PASSWORD_' | sudo openconnect --passwd-on-stdin --protocol=anyconnect --user="_YOUR_IBM_EMAIL_ADDRESS_" sasvpn06.emea.ibm.com/gettingstarted
     ```

**NOTE 1: Approximately every minute or so, you will see a message in the same console as the python3 command saying "vpn ERROR: No device found with hostname '*YOUR_HOSTNAME_HERE*'". This is the expected output during the registration process and indicates the polling of the server to find out whether your machine has been registered yet is working. These messages will stop when your device is registered (or if you close the terminal).

**NOTE 2: The registration script will fail with the error "Connection failed. Check your credentials or network connection" until such time as your device has been picked up by BigFix and appears on the [SASVPN User Devices Page](https://w3-01.ibm.com/tools/vpn/enduser/userdevice.php)**. This can take anywhere up to 24 hours. After your device appears in BigFix, registration will be possible.

Once successfully registered, your device should appear in https://devices.w3cloud.ibm.com/devices/#/myDevices and you should have your [SAS certificate](https://w3-01.ibm.com/tools/vpn/enduser/userdevice.php) downloaded and configured automatically for you.



## Tuning

### Enable simple password

```
Executing: /usr/bin/authselect check
Executing: /usr/bin/authselect current --raw
Executing: /usr/bin/authselect select sssd --force
minlen=15
fr054721:~$ echo "abc123" | sudo passwd root --stdin -f
Changing password for user root.
passwd: all authentication tokens updated successfully.
fr054721:~$ echo "spcspc" | sudo passwd root --stdin -f
Changing password for user root.
passwd: all authentication tokens updated successfully.
fr054721:~$ echo "spcspc" | sudo passwd fr054721 --stdin -f
Changing password for user fr054721.
passwd: all authentication tokens updated successfully.
```



### Manage Microsoft Defender ATP

```
sudo systemctl stop mdatp
```



### Manage Tivoli Endpoint Manager

```
sudo systemctl stop besclient
```

