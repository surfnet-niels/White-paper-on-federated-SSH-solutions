ssh-oidc
--------
[ssh-oidc](https://github.com/EOSC-synergy/ssh-oidc) is a collection of
tools that enable ssh using OpenID-Connect access tokens.

Most of the individual tools work standalone, but they may be combined to
establish complex installations with dynamic account provisioning in
multi-OP and multi-VO contexts.

The tools share these common design criteria:
- None of the ssh-client or ssh-server components will be modified
- Do not store state whenever possible
    - Single exception: federated-user -> local-user mapping is stored in `passwd`
- Small components that work individually (one tool for one job)

For demonstrating the current state of the approach, all tools have been
combined into a demo installation at <https://ssh-oidc-demo.data.kit.edu>

The individual tools used to compose the solution are:

- pam-ssh-oidc: a PAM module developed by PSNC, that adds the "Access
  Token" prompt to the list of supported authentications. This allows ssh
  login, using a pre-created user, using the access token as a password.
- [motley-cue](https://motley-cue.readthedocs.io/en/latest/) is a
  server-side tool which receives OIDC-tokens via REST-API, validates them
  and provisions users locally or in LDAP. It acts as an interface to
  provide advanced features such as
    - user registration flow (on demand, optional)
    - user provisioning (on demand, optional, based on assurance and
      external authorisation sources (e.g. VOs))
    - one-time password support (for access tokens longer than 1024 byte)
    - user deprovisioning / suspension in case of security incident
- [mccli](https://mccli.readthedocs.io/en/latest) is a wrapper for SSH
  which performs the initial API calls to transmit the token and then
  connects via SSH. It is the client-side tool that simplifies the
  interaction with motley-cue and oidc-agent. It supports `ssh`, `scp`,
  and `sftp`
- [oidc-agent](https://github.com/indigo-dc/oidc-agent): a client-side
  tool for obtaining OIDC access tokens

The different tools and the points they address are shown in the table:

|                                  | pam-ssh-oidc | motley-cue | mccli | oidc-agent |  | OIDC-OP | External VO mgmt |
|----------------------------------|--------------|------------|-------|------------|--|---------|------------------|
| Know your username               | x            |            |       |            |  |         |                  |
| Provision users                  |              | x          | x     |            |  |         |                  |
| Offline flow for user provision  |              | x          |       |            |  |         |                  |
| Access Token > 1k                |              | x          | x     | x          |  |         |                  |
| Obtain ATs                       |              |            |       | x          |  |         |                  |
| Stop further authentications     |              |            |       |            |  | x       | x                |
| Trigger deprovision / suspension |              | x          |       |            |  |         |                  |
| Authorisation                    |              | x          |       |            |  | x       | x                |
| Delegation / token forwarding    |              |            | x     | x          |  |         |                  |


## Overall Architecture

The architecture diagramme shows how all of the above tools may work
together, if desired. The diagramme shows the `ssh-client`, the
`ssh-server`, as well as the OIDC Provider.


![architecture](/images/ssh-oidc-kit-arch.png)

## Login Flow

The following steps take place when a user logs in to the ssh server:

1. In this step the user has multiple options:
    1. The `mccli` tool may use `oidc-agent` to obtain an access token
       from the OIDC Provider (OP). Alternatively, the access token can be
       provided in an environment variable, e.g. `OIDC`.
    1. The user may directly use any ssh-client. She will need to know the
       username on the remote host, and will be queried for access token
       instead of a password.
1. When used, `mccli` contacts the `motley-cue` daemon (on or nearby the
   ssh server). It makes sure that the user exists (if authorised), and
   generates a one-time-password, in case the access token is too long for
   the ssh client used (openssh has a limit of 1k).<br/> `motley-cue` will
   update the user's groups (based on entitlements) and/or create the user
   if the account didn't exist already. (Pre-existing accounts will be
   used whenever available, local-remote user mapping is stored in e.g.
   `/etc/passwd` or similar, depending on the backend).
1. The ssh-client performs a standard ssh authentication flow. As part of
   this the access token is passed to `pam-ssh-oidc`, which either
   verifies the token with the OIDC-Provider, or locally with
   `motley-cue`.<br/>
   If configured, other `pam` modules may be called within this login
   process. This may include prompting for a `password`, or requesting a
   `second factor`.
1. On success the user is given the configured login shell.

## Advanced Features

In addition to the above flow, the following features are available.

- *Dynamic user provisioning*: When using `motley-cue`, user-accounts may
    be dynamically provisioned. Policies for user-naming and
    group-creation can be configured and are very flexible. Supported
    backends include `local_unix`, `ldap`, and a REST interface to the KIT
    `reg-app` system.
- *Offline provisioning*: Some site policies (HPC in particular) require
    the manual inspection of provisioning requests. This is supported by
    informing administrators (e.g. by email) about a new-user registration
    event. The user will then be able to login, after the admin approved
    the request.
- *Assurance*: Authorisation of user access can (in addition to
    group membership or entitlements) be based on the assurance level of
    an incoming user. `motley-cue` support for assurance is flexible
    enough to support arbitrary assurance statements. Examples for the
    REFEDS Assurance Framework are included.
- *Agent forwarding*: Just as with `ssh-agent`, `oidc-agent`
    supports remote sockets. When using `mccli`, the ssh-client is called
    with the required parameters to forward a remote socket to the
    client-host, so that fresh access tokens are available for processes
    on the server side. When not using `mccli`, the required parameters
    may easily be added to the ssh command line.
- **Security Incident Support**: `motley-cue` supports an `admin` endpoint
    which allows authorised administrators (identified via OIDC
    `sub`+`iss`) to suspend or resume user accounts. This is intended for
    enabling CERT staff to quickly mitigate security incidents.

## Installation

We provide packages for a large set of linux distribution (Debian, Ubuntu,
Centos, Rockylinux, OpenSuSE, Archlinux, ...) for client and server packages.

MacOS and Windows are additionally supported with client packages.

All packages (except MacOS currently) are available via
<https://repo.data.kit.edu>

The (optional) client-side packages (`oidc-agent` and `mccli`) are
additionally part of most Debian-, and Arch- based distributions
(`oidc-agent`) or easily installable via `pip install` (`mccli`).

## Configuration

The system is configured via the following configuration files.
(`oidc-agent` is considered out of scope for this documentation.)

### `/etc/pam.d/sshd`

A single line in the beginning of `/etc/pam.d/sshd` is sufficient to
enable `pam-ssh-oidc`:
```config
auth   sufficient pam_oidc_token.so config=/etc/pam.d/pam-ssh-oidc-config.ini
```
This line is automatically added when using the `pam-ssh-oidc-autoconfig`
package.

The file `/etc/pam.d/pam-ssh-oidc-config.ini` configures how the PAM module verifies the token and username. For example, to delegate the verification to the local `motley-cue` daemon, the following configuration can be used:

```config
[user_verification]
local = false
verify_endpoint = http://localhost:8080/verify_user
```

This configuration is also automatically added when using the `pam-ssh-oidc-autoconfig` package.

### `/etc/ssh/sshd_config`

SSHD should be configured to accept `KbdInteractiveAuthentication`
(previously `ChallengeResponseAuthentication`):

```config
ChallengeResponseAuthentication yes
KbdInteractiveAuthentication yes
```

### `/etc/motley-cue/motley-cue.conf`

Here we configure several different aspects:

- Which OPs do we trust
- Authorisation:
    - Which Virtual Organisations do we support
    - Which individual users do we support
- The privacy statement to display
- Authorised security staff

The default self-documenting configuration file is shipped with the
`motley-cue` installation.

### `/etc/motley-cue/feudal.conf`

Feudal adapter is the plugin-based tool that implements user provisioning.
Aspects that are configured here include:

- Minimally required levels of assurances
- How to map remote users to local unix accounts
- How to map VO-memberships to local unix groups
- Which backend to use

Again, the configuration file shipped with the `motley-cue` installation
is self explanatory.


### Discussion of Evaluation Criteria

- Does the solution mitigate sharing of SSH keys?
    - We replace the need for keys by using OIDC-Access tokens. SSH keys
      may still be used, but are no longer necessary
- What are the client requirements and supported platforms?
    - Clients not require any installation. However, we suggest the
      installation of `oidc-agent` and `mccli` (windows: our `putty
      plugin`) for improved user experience. All packages are available
      for most linux distributions, macOs and windows.
- What are the SSH server requirements and does the solution require
  additional software beyond SSH server?
    - The server requires `pam-ssh-oidc`, and strongly benefits from
      `motley-cue`. Packages exist for all major linux distributions in
      `apt`, `yum`, or `zypper` repositories at
      <https://repo.data.kit.edu>
- Does the solution allow for non interactive client logins?
    - Sure
- Does the solution allow for delegation?
    - Sure: with oidc-agent forwarding, you can use fresh access tokens on the
      remote host for further ssh-oidc-logins, REST calls or other purposes.
- What requirements are put on the incoming federated identity?
    - Configurable. We suggest RAF `cappuccino` and membership in a VO.
- How is provisioning towards the SSH server set up?
    - user provisioning can be handled by `motley-cue` via `feudal`. Offline
      provisioning (with admin approval) and dynamic provisioning (of authorised
      users) are supported, with configurable user naming and group creation policies.
    - implemented as an interface to any local account management system with a plugin
      architecture; supported backends: `local_unix`, `ldap`, the KIT `reg-app` system.
- How does revocation work?
    - Authorised personnel may use the `/admin` endpoint of `motley-cue`
      which we expose on the server side
    - Individual tokens can also be revoked by the user
- Does the setup allow for MFA
    - Yes
- Any provisions for mitigating server TOFU
    - ???


