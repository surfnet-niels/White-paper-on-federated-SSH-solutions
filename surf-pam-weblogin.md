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


