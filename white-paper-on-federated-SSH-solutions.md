The goal of this paper is to present and evaluate different ways in which federated identity – as known on the web – could be leveraged for SSH access to resources.

  

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

\- How does the solution mitigate sharing of SSH keys?  
\- What are the client requirements and supported platforms?  
\- What are the SSH server requirements and does the solution require additional software beyond SSH server?  
\- What requirements are put on the incoming identity?  
\- How is provisioning (if required) towards the SSH server set up?  
\- How does revocation work?  
\- Does the setup allow for MFA?

Solution descriptions
=====================

The DeiC "SSH Certificates in a federated world" solution – proof of concept
----------------------------------------------------------------------------

For at thorough description of SSH certificates see the above mentioned blog post from SmallStep. Here is a very short introduction:

SSH supports pubkey authentication for both servers (as the only method) and clients/users (among other methods). A SSH certificate is, for all practical purposes, a SSH public key with some additional information, signed by a SSH certificate authority.

A SSH certificate is encoded according to rfc4251, i.e. using the same encoding as the SSH protocol itself. Only 6 data type representations are used (byte, boolean, uint32, uint64, string and mpint), making it simple to (un)marshal a certificate in any programming language.

In general, certificates makes it possible for a SSH server to trust users from a number of SSH certificate authorities just by listing the certificate authority's public key and vice versa for a user to trust hosts just by enumerating the public keys of the trusted certificate authoritys in the .ssh/known\_hosts file.

In addition the the public key a certificate contains:

1.  Identity information of the user (KeyId)
2.  Identity/role/hostname information (ValidPrincipals)
3.  Validity period information (ValidAfter, ValidAfter)
4.  Critical Options – information that can't be ignored by a SSH server
5.  Extensions – general extension information

This means that the certificate can be used as holder of key token by a ssh server i.e. the additional information can be used to make authorization decisions on the SSH server.

The ValidPrincipals list is used in very large deployments where where there is only one user created on a server. A sysadmin's certificate then contains the roles she can log in with and the the server has access to a list - a configured list or one provided by a AuthorizedPrincipalsCommand that must contain on of the users principals for the user to be able to login. Even though all users then operate under the same username the actual user (KeyId) is logged on login.

As a default the existence of a username in the ValidPrincipals list allows that user to login.

The ValidAfter/ValidBefore fields allows the SSH certificate authority to issue short-lived certificates according to it's policy so that a revocation mechanism is not needed.

This project uses the Extensions field to let the SSH certificate authority add information from the assertion provided by a federated login to the certificate - as well as additional information available to the certificate authority e.g. from an attribute authority.

In the following, SAML federation terminology is used, because the DeiC SSH certificate authority uses SAML, but any federation technology can be used – a SSH certificate authority can even be an IdP in its own right.

So specifically the identity validation that is the basis for registering a SSH credential, i.e. a public key, is based on the identity of the user the from the SAML assertion.

Ssh servers that trust one or more ssh certificate authority's are then able to

\- let users login based on their identity stated in their certificate, if they are already registered and maintained by other means

\- Register new users and keep information on existing users updated based on both their identity and the additional information e.g. group membership from the certificate authority

### The Certificate Authority

The method the Deic SSH certificate authority uses for issuing certificates is as follows:

1.  The certificate authority is a SAML service provider that, when it receives an SAML assertion saves it temporarily under a random name
2.  The name is made available to the user as a parameter of a ssh command string that the user can copy from the certificate authority's webpage and paste in a terminal
3.  The ssh command connects to the SSH certificate authority process on a port where it listens for ssh connections
4.  The SSH certificate authority now has access to the users public key - as part of the ssh authentication protocol it is made available to the ssh server
5.  A successful check of the signature of the ssh session\_key with the public key proves that the user is in possession of the corresponding private key
6.  The SSH certificate authority also has access to the SAML assertion, as the random name was sent as a parameter in the ssh connection
7.  It can then create a ssh certificate using the user's public key, the available information and it's policy for for eg. the validity time for the certificate and potentially a schema for mapping the user's SAML identity to a unix username
8.  The issued certificate is sent back to the client via the ssh connection to the ssh CA and saved where ssh expects to find it.

As long as the certificate is valid it can then be used as the users public key. The user will not experience any difference and can continue to use ssh - both interactively and in the background.

### Sshd Configuration - existing users

If the ssh server only needs to use the certificate as an identity token for recognizing an existing user the configuration of the server is simple. The relevant part of the sshd configuration is:

TrustedUserCAKeys /etc/ssh/sshd\_config.d/ca-keys.pub

Tells sshd where to look for a file containing lines of public keys (not certificates) for trusted SSH certificate authorities

AuthorizedKeysFile none

Tells sshd not to use "normal" AuthorizedKeysFiles for authenticating users. If not set i would be possible for users to upload their public key to the ".ssh/authorized\_keys" and thus bypass the limitations of the certificates.

In this simple case the principals must contain the username of the user.

### Sshd Configuration - user management on-the-fly

  
if the ssh server in addition to using the certificate as an identity token also wants to employ it for creating and maintaining users "on the fly" using the information in the certificate it can be done as follows:

Add the following line to the sshd configuration:

ExposeAuthInfo yes  
AllowUsers <name of special user>

This makes the ssh certificate used for authentication available to the user's shell - via an sshd provided environment variable - SSH\_USER\_AUTH - that contains the name of a file with the certificate.

Then let the users login as a special user with rights to do user management and a special shell (the repository for the DeiC SSH CA contains one).

On login the special shell creates/manages the user based on the information in the certificate and then switches to the actual user.

### Server certificates

This project has only looked at providing user certificates. Issuing host certificates requires a different governance setup than a user federation.

### Historic notes

Early versions of the on-the-fly user management used the sshd facility AuthorizedPrincipalsCommand. This allows an sshd administrator to configure a command that - with access to the certificate - issues a list of authorised principals. As the command has access to the certificate it was believed that it could be used to create/manage users on the fly. But it appears that the command is called to early in the sshd authentication process to trust the certificate is actually used for the authentication. It can thus not be used for usermanagement.

The principals system is used in very large deployments where where there is only one user created on a server. A sysadmin's certificate then contains the roles she can log in with and the the server has access to a list - a configured list or one provided by a AuthorizedPrincipalsCommand that must contain on of the users principals for the user to be able to log n. Even though all users then operate under the same username the actual user is logged on login.

### Discussion of Evaluation Criteria

*   Does the solution mitigate sharing of SSH keys?
    *   Even if a private key is “shared” or stolen login requires a “recent” certificate based on a federated login. I.e. it requires something based an a persons institutional identity, which we doubt will be “shared”
*   What are the client requirements and supported platforms?
    *   An openSSH ssh client
*   What are the SSH server requirements and does the solution require additional software beyond SSH server?
    *   An openSSH ssh server (ssh). No
*   Does the solution allow for non interactive client logins?
    *   Yes - in this context a certificate is just a time limited public key. Also works with subsystems like sftp and scp"
*   Does the solution allow for delegation?
    *   Yes, standard ssh delegation
*   What requirements are put on the incoming federated identity?
    *   None - but some form for username coordination is needed if multiple certificate authorities are trusted
*   How is provisioning towards the SSH server set up?
    *   Depends on the situation, but a certificate can contain information to do front end ad-hoc creation of users. We have a working prototype for that.
*   How does revocation work?
    *   Just use a short validity time for the certificate.
*   Does the setup allow for MFA
    *   Apart from the obvious mfa from the federated login, the private ssh key can be mfa’ed.
*   Any provisions for mitigating server TOFU
    *   Yes, use host certificates. This is possible independent of the solution for users.

PAM-weblogin
------------
[https://github.com/SURFscz/pam-weblogin](PAM-weblogin) is a PAM module written in C that enables a challenge-response login flow using SSHd keyboard-interactive login.
The idea is to authenticate users based on the preprovisioned username and public key and then drop into PAM to check whether the user can perform a federative login on the challenge (unique URL) that is presented in the terminal. After clicking the URL, the user is expected to login (federatively) and on successful return a verification code is shown that needs to be supplied in the interactive session before continuing. If the code and login actor match the federative account based on a configurable claim, PAM exits with PAM_SUCCESS and the SSH login continues.

![image](https://user-images.githubusercontent.com/1901782/203533315-018d1ee2-245c-47e7-bc99-00ffa6bc39d6.png)
  
### Smart Shell experiments
[https://github.com/SURFscz/pam-weblogin/tree/main/shell](PAM-weblogin/shell) is an ongoing experiment to use a so-called Smart Shell to do the keyboard-interactive part of the login. A big advantage is that you are not confined to a strict C solution (it is implemented in python) and that the SSH requirement that the user must exist can be forgone. This opens up the path for a functional federative login account, which finally drops into the shell of the authenticated federative user, possible just right after the account has been provisioned.

![image](https://user-images.githubusercontent.com/1901782/203533425-f4154b26-bbf1-4f59-ad17-b604d00fa079.png)
  
* How does the solution mitigate sharing of SSH keys?
  * A federative authentication is required to confirm the login. The PAM module can even be used without SSH key (as a single factor, not recommended).
* What are the client requirements and supported platforms?
  * The client is an unmodified ssh client.
* What are the SSH server requirements and does the solution require additional software beyond SSH server?
  * The module needs to be compiled on the server or a package can be created. The module itself consists of a single file that needs to be copied in place. Of course, necessary changes need to be made to SSHd and PAM configuration(s).
* What requirements are put on the incoming identity?
  * The identity needs to exist, but experiments are in place to replace the PAM module with a Smart Shell, which could do JIT provisioning of the user just before handing over the shell to the authenticated user.
* How is provisioning (if required) towards the SSH server set up?
  * Any form of (pre) provisioning will work. Specifically in SRAM, we use LDAP provisioning.
* How does revocation work?
  * The federative login will fail.
* Does the setup allow for MFA?
  * Yes, e.g. as a step-up requirement of the federative login flow.


Technical details - BETTER NAME NEEDED
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

usePAM yes

Set this to 'yes' to enable PAM authentication, account processing, and session processing. If this is enabled, PAM authentication will be allowed through the KbdInteractiveAuthentication and PasswordAuthentication. Depending on your PAM configuration, PAM authentication via KbdInteractiveAuthentication may bypass the setting of "PermitRootLogin without-password". If you just want the PAM account and session checks to run without PAM authentication, then enable this but set PasswordAuthentication and KbdInteractiveAuthentication to 'no'.  

KbdInteractiveAuthentication yes  

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
