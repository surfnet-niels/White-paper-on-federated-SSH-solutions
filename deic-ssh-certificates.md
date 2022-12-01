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

* let users login based on their identity stated in their certificate, if they are already registered and maintained by other means
* Register new users and keep information on existing users updated based on both their identity and the additional information e.g. group membership from the certificate authority

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
```
ExposeAuthInfo yes
AllowUsers <name of special user>
```
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

