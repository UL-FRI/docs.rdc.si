# Obtaining an account

Access to the FRIDA compute infrastructure is granted (upon request) to all employees of UL FRI. Under the guidance and responsibility of a UL FRI employee access can be granted also to students who cooperate on research lab projects, or require access to the infrastructure to work on their doctoral, master's or diploma theses. The access request must come from a UL FRI employee (supervisor), stating the research lab, project, and/or thesis topic, as well as the duration for which access is requested.

Requests for access, as well as technical questions, should be directed to frida@rdc.si.

## Stale accounts

Usage of FRIDA is monitored to ensure fair sharing of the available resources. Stale accounts, i.e. accounts that did not submit any jobs in the last 6 months, are automatically disabled, and the account-associated data is archived. If the account was the last account of a specific research lab, group, or project, the corresponding research lab, group, or project data is archived as well.

The archived data is kept for 3 additional months to facilitate an eventual reenablement of the account or download of the associated data upon request. After this grace period, all data is permanently deleted.

## Passwordless access

In line with the UL security policies, access to FRIDA is strictly multi-factor authentication (MFA) based. To achieve the most seamless integration and provide the best possible user experience, FRIDA opted for Teleport (see [How Teleport Works](https://goteleport.com/how-it-works/)) and the corresponding `tsh` Command Line Interface (CLI) client (see [Using the tsh Command Line Tool](https://goteleport.com/docs/connect-your-client/introduction/)).

For the best user experience, we suggest users set up their accounts for passwordless access via Apple Touch ID, Windows Hello or [Yubikey BIO](https://www.yubico.com/si/product/yubikey-bio-series/yubikey-c-bio/). A still valid alternative is 2FA via [Youbikey 5C Nano](https://www.yubico.com/si/product/yubikey-5c-nano/) or OTP with [Raivo](https://raivo-otp.com).

## Registration

Once granted access to your request, you will receive a Teleport signup link. Open the link and follow the instructions to register a password and OTP device. You can also register a hardware key (Apple Touch ID, Windows Hello, or Yubikey BIO) for passwordless web access. Note, however, that you will need to register the hardware key for every browser that you intend to use (this is by design, a security requirement of hardware keys).

!!! warning
    Registration of an OTP device during the registration process is mandatory, otherwise you will later not be able to register a hardware key for CLI access.

Once the registration is finished follow the official Teleport instructions to install the Teleport Community Edition of the `tsh` CLI client appropriate to your OS ([macOS](https://goteleport.com/docs/installation/#macos), [Windows](https://goteleport.com/docs/installation/#windows-tsh-client-only), or [Linux](https://goteleport.com/docs/installation/#linux)). If you wish to do so, you can, as an addition, install [Teleport Connect](https://goteleport.com/docs/connect-your-client/teleport-connect/) (a Graphical User Interface (GUI) desktop client), but for most use cases this is not necessary. Ensure that the `tsh` client install location is included in your `PATH` variable.

Now you can register your hardware key (Apple Touch ID, Windows Hello, or Yubikey BIO) to enable passwordless infrastructure access also via CLI. To do so, you execute the following command in your terminal. For successful registration, you will need to provide your password and second-factor key (OTP).
```bash
$ tsh --proxy=rdc.si --user={username} mfa add --type=TOUCHID --name=touchid.cli
```

<!--
**Kako je z več browserji?, Kako je z registracijo mfa v CLI?**

*_Note that on Apple you have to install the signed ???_

_kako vzpostaviti passwordless, in kako registrirat 2FA via 5C Nano (tudi OTP z Raivo / google auth ...?)_
-->

## First access

Use the `tsh` client to authenticate yourself to the FRIDA Teleport gateway
```bash
$ tsh --proxy=rdc.si --user={username} login
```

Upon successful authentication create or edit the SSH config file corresponding to your OS (`~/.ssh/config` on Unix-like, `%userprofile%/.ssh/config` on Windows systems), and add to the top the snippet that you obtain by running the command `tsh config`. While doing so, add the following lines to the end of the section marked as `# Common flags for all ... hosts`.
```bash
ServerAliveInterval 300
ServerAliveCountMax 2
TCPKeepAlive yes
UseKeychain yes
ForwardAgent yes
AddKeysToAgent yes
```

Save the SSH config and you are set to go. From now on you can interact with FRIDA by running standard `ssh` and `scp` commands.

## VSCode and Remote SSH access to the login node

Visual Studio Code's Remote SSH extension (part of the [Remote Development Extension pack](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack)) in the Remote Explorer does not list wildcard-based hosts declared in the SSH config file. To have the FRIDA login node available in Remote Explorer you must add to the SSH config file also the following snippet.
```bash
# FRIDA login host for VSCode Remote SSH
Host login-frida
    Hostname login-frida.rdc.si
    User {username}
    Port 3022
    ProxyCommand "/usr/local/bin/tsh" proxy ssh --cluster=rdc.si --proxy=rdc.si:443 %r@%h:%p
```
