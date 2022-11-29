ssh-oidc
--------
[ssh-oidc](https://github.com/EOSC-synergy/ssh-oidc) is a collection of
tools that provide enable ssh, using OpenID-Connect access tokens.

Most of the individual tools work standalone, but they may be combined to
establish complex installations with dynamic account provisioning in
multi-OP and multi-VO contexts.
None of the ssh-client or ssh-server components are modified.

For demonstrating the current state of the approach, all tools have been
combined into a demo installation at <https://ssh-oidc-demo.data.kit.edu>

The individual tools used to compose the solution are:

- pam-ssh-oidc: a PAM module developed by PSNC. That PAM module allows ssh
    login, using a pre-created user, and the access token as a password.
- [motley-cue](https://motley-cue.readthedocs.io/en/latest/): a REST interface for the ssh server, that acts as an
    interface to provide advanced features such as
    - user registration flow (on demand, optional)
    - user provisioning (on demand, optional, based on assurance and external authorisation sources (e.g. VOs))
    - one-time password support (for access tokens longer than 1024 byte)
    - user deprovisioning / suspension in case of security incident
- [mccli](https://mccli.readthedocs.io/en/latest): a client-side tool to simplify interaction with motley-cue and
    oidc-agent. Supports `ssh`, `scp`, and `sftp`
- [oidc-agent](https://github.com/indigo-dc/oidc-agent): a client-side tool for obtaining OIDC access tokens





### Discussion of Evaluation Criteria

* Does the solution mitigate sharing of SSH keys?
* What are the client requirements and supported platforms?
* What are the SSH server requirements and does the solution require additional software beyond SSH server?
* Does the solution allow for non interactive client logins?
* Does the solution allow for delegation?
* What requirements are put on the incoming federated identity?
* How is provisioning towards the SSH server set up?
* How does revocation work?
* Does the setup allow for MFA
* Any provisions for mitigating server TOFU


