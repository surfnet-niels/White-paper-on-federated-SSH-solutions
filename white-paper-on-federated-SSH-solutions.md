The goal of this paper is to present and evaluate different ways in which federated identity – as known as webSSO – could be leveraged for SSH access to resources.



Problem Description
===================

Researchers sometimes want to access services over SSH. These services belong to an organization that does not necessarily have a (direct) relationship with that particular researcher, i.e. there is no trusted path for the researcher to tell the organization about their public SSH key. So "Federated SSH" aims at solving this problem, which consists of a number of parts:

1.  How to teach the SSH server which users (with matching SSH public key) to accept?
2.  How to inform the researchers what the server's/servers' SSH host key(s) is/are, to avoid relying on "Trust On First Use" (TOFU)?
3.  How to _revoke_ access for a particular user if they are no longer a researcher in good standing, left the project, or left their institute?

Why
===

Why do we want to have a solution for this particular problem? Solving the above problems requires a lot of work, especially when dealing with a great number of researchers, or servers. Manually collecting SSH public keys from authorized users, making sure they belong to the user, and also figuring out when the user is no longer allowed to access the service is (quite) difficult.

See [https://smallstep.comblog/use-ssh-certificates/](https://smallstep.com/blog/use-ssh-certificates/) .

Federated SSO, on the other hand, scores well on the above criteria (User experience, scaling up, security) but is usually limited to the web.

In this white paper we present different solutions that could be used to leverage the advantages of federated web SSO solutions for SSH access management.



Introduction into 'Web SSO'
===========================

*   federated login and SAML login as we use it in eduGAIN / national fed
*   explain context of why using fed login or user management
*   explain context about aarc BPA model

Comparison chart
================

Here we will compare the different solutions presented in the 2nd workshop demo to the requirements:

* How does the solution mitigate sharing of SSH keys?
* What are the client requirements and supported platforms?
* What are the SSH server requirements and does the solution require additional software beyond SSH server?
* What requirements are put on the incoming identity?
* How is provisioning (if required) towards the SSH server set up?
* How does revocation work?
* Does the setup allow for MFA?

Solution descriptions
=====================

- [The DeiC "SSH Certificates in a federated world" solution – proof of concept](deic-ssh-certificates.md)
- [The KIT "OpenID Connect SSH" toolbox](ssh-oidc-kit.md)
- [The SURF "PAM-weblogin](surf-pam-weblogin.md)
- [The AWI Smartshell with krest](awi-smartshell.md)
- [OAuth Authorization for SSH](https://github.com/XSEDE/oauth-ssh/)



Appendix
======================================

SSH Daemon and it's points of interception
------------------------------------------

Originally, users logged in to an ssh session using username/password combination. Most users/servers though, have switched to PKI (public/private keypairs) to set up a session and nowadays, the private key can even be stored on a FIDO2 compatible hardware token.

However, a third way of authentication and authorization can be obtained from logging in with a ssh user certificate. An SSH certificate is a normal public key, that is augmented with user claims, signed by a certificate authority, which is ultimately trusted by the server. The ssh server can be configured to only trust certificates (not the public keys) and use the contained claims to make authorization decisions and elevate the user priviliges based on the role or other claims contained within the certificate. Take note that the public-key part of the user is still used to authenticate the user, but the additional signature and claims of the CA server are used to make further authorization decisions.

Below a schematic representation of the key material and where it resides/orginates from


SSH Certificates
----------------

Based their existing work, WAYF explains in this placeholder what's great about SSH certificates

![image](https://user-images.githubusercontent.com/1901782/203533666-ae8fb14e-9ae5-42f1-ad6e-ec49e571bb06.png)

PAM
---

SSHd can be configured to leverage PAM authentication and support the PAM conversation functionality through KbdInteractiveAuthentication:
```
usePAM yes
```
Set this to 'yes' to enable PAM authentication, account processing, and session processing. If this is enabled, PAM authentication will be allowed through the KbdInteractiveAuthentication and PasswordAuthentication.

Depending on your PAM configuration, PAM authentication via KbdInteractiveAuthentication may bypass the setting of "PermitRootLogin without-password". If you just want the PAM account and session checks to run without PAM authentication, then enable this but set PasswordAuthentication and KbdInteractiveAuthentication to 'no'.
```
KbdInteractiveAuthentication yes
```
[KbdInteractiveAuthentication](https://www.rfc-editor.org/rfc/rfc4256)

Change to yes to enable challenge-response passwords (beware issues with some PAM modules and threads)

In the (sshd) PAM configuration, PAM module can be added and configured to be either _required_ or _sufficient_ or even more complex conditional access policies.

Below is the flow diagram of the [pam-weblogin](https://github.com/SURFscz/pam-weblogin/) module that provides the requirement of loging in to a website that is printed on the challenge conversation during login:


### PAM Caveats

As SSHd checks the existance of the login user (the real user trying to login, or the user before the @ in the ssh location, it is impossible to force web-based login based on a generic fake user (e.g. web@service.example.com). Even if this was possible, SSHd does not accept a changed user identifier after completing the PAM login, so the effective ssh user can not be overriden by PAM.

Drop into Smart Shell
---------------------

Instead of letting PAM do the heavy-lifting of the authorization process, we can also let sshd check a key/certificate or password and then hand over the session to an executable instead of shell that is specially crafted to do some extra authorization checks and then hand over the login to the appropriate user by sudo'ing to this user and starting a real shell. The advantage is that we can now login using a functional account (as long as the normal authentication with key or password works) and then become the new authorized user.

To make the functional account weblogin recognise and accept public keys for all valid users of the service, the sshd options AuthorizedKeysCommand and accompanying AuthorizedKeysCommandUser can be used to point to an executable that generates a list of authorized keys, collected from a central store of keys like an MMS or LDAP directory.

[https://github.com/SURFscz/pam-weblogin/tree/main/shell](https://github.com/SURFscz/pam-weblogin/tree/main/shell)
```
$ ssh weblogin@weblogin
Linux weblogin 5.10.0-15-amd64 #1 SMP Debian 5.10.120-1 (2022-06-09) x86\_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/\*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Oct 18 14:48:52 2022 from 192.168.56.1
Hello weblogin. To continue, visit [http://localhost:5001/pam-weblogin/login/k4vI3DOT](http://localhost:5001/pam-weblogin/login/k4vI3DOT) and enter verification code
code:
myshell@weblogin:~$ id
uid=1002(myshell) gid=1002(myshell) groups=1002(myshell)
myshell@weblogin:~$
logout
Connection to weblogin closed.
```
systemd
-------

Bits and pieces on how systemd can help resolving system users.
