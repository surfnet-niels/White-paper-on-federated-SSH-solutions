ssh-oidc
--------
[ssh-oidc](https://github.com/EOSC-synergy/ssh-oidc) is a collection of
tools that provide enable ssh, using OpenID-Connect Access Tokens.

Most of the individual tools work standalone, but they may be combined to
establish complex installations with dynamic account provisioning in
multi-OP and multi-VO contexts.

For demonstrating the current state of the approach, all tools have been
combined into a demo installation at <https://ssh-oidc-demo.data.kit.edu>

The individual tools used to compose the solution are:

- pam-ssh-oidc: a PAM module developed by PSNC. That PAM module allows ssh
login, using a pre-created user, and the Access Token as a password.




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


