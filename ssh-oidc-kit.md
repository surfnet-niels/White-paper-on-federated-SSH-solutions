ssh-oidc
--------
[ssh-oidc](https://github.com/EOSC-synergy/ssh-oidc) is a collection of
tools that provide enable ssh, using OpenID-Connect access tokens.

Most of the individual tools work standalone, but they may be combined to
establish complex installations with dynamic account provisioning in
multi-OP and multi-VO contexts.

The tools share these common design criteria:
- None of the ssh-client or ssh-server components will be modified
- Do not store state whenever possible
    - Single exception: federated-user -> local-user mapping is stored in `passwd`
- Small components that work individually (one tool for one job)
- 

For demonstrating the current state of the approach, all tools have been
combined into a demo installation at <https://ssh-oidc-demo.data.kit.edu>

The individual tools used to compose the solution are:

- pam-ssh-oidc: a PAM module developed by PSNC, that adds the "Access
  Token" prompt to the list of supported authentications. this allows ssh
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
  which performs the initial API call to transmit the token and then
  connects via SSH. It is the client-side tool that simplifie the
  interaction with motley-cue and oidc-agent. It supports `ssh`, `scp`,
  and `sftp`
- [oidc-agent](https://github.com/indigo-dc/oidc-agent): a client-side
  tool for obtaining OIDC access tokens


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


