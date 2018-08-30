# LAPSforMac -- updated compatibility for High Sierra

Huge, **huge** kudos to Phil Redfern and the University of Nebraska for this script. I simply updated the LAPS script to resolve some quirks with the `jamf_binary` in High Sierra, resulting in odd behavior regarding secureToken and FileVault enablement on Active Directory-bound accounts. 

The LAPSforMac script works on local accounts as well, my environment just happens to bind to AD during user account provisioning.


## `dscl`, secureToken, and FileVault

* The original LAPS script uses `$jamf_binary` to reset the user password, which in turn utilizes `dscl`
* `dscl` technically performs a successful password reset, however it exhibits the following behavior in macOS 10.13.6 when a Jamf policy executes the LAPS script on an AD-bound machine:
  
### Scenario 1
  
* A local admin account with secureToken is currently logged in. The local admin logs out.
* AD-user logs in
* A secureToken dialog window appears to request an admin's credentials to enable secureToken for the AD-user
* secureToken is successfully enabled for the AD-user
* Run `sysadminctl -secureTokenStatus ADUserHere` to confirm secureToken is ENABLED
* A Jamf policy silently changes the local admin user in the background using the LAPS script
* The AD-user either logs out to trigger a deferred FileVault enablement, or the AD user visits System Preferences to enable FileVault
* An error appears stating FileVault could not be enabled
* Reboot and log in as the local admin user
* Run `sysadminctl -secureTokenStatus LocalAdminUserHere` to confirm secureToken is ENABLED for the local admin
* Run `sysadminctl -secureTokenStatus ADUserHere` and discover secureToken is DISABLED for the AD-user, despite "successfully" gaining a secureToken during the initial AD-user login
* Attempt to enable FileVault via System Preferences or `fdesetup` while logged in as the local admin, receive same error
* Log out of local admin account, deferred FileVault enablement dialog window appears, receive same error
* You're in a bad situation, because even though `sysadminctl` says the local admin has a secureToken, `dscl` somehow strips this attribute from both the local admin & AD-user when the Jamf policy ran the LAPS script.

### Scenario 2

* Before the local admin account is able to sign in for the first time (created via PreStage Enrollment, SetupAssistant, or pkg/script), a Jamf policy silently changes the local admin password in the background using the LAPS script
* The local admin signs in using the new LAPS password
* Run `sysadminctl -secureTokenStatus LocalAdminUserHere` to find that secureToken is DISABLED
* It appears that because `dscl` changed the user password before the first login, it somehow strips the secureToken attribute from the first local admin user account signing in
* This puts you in a bad situation, because if the sole admin account on the machine has no secureToken, there's no way to create an additional user with secureToken, or enable FileVault


## Resolution: use `sysadminctl`

Replace `$jamf_binary resetPassword -username $resetUser -password $newPass`

with

`sysadminctl -adminUser $resetUser -adminPassword $oldPass -resetPasswordFor $resetUser -newPassword $newPass`

In addition: because `sysadminctl -resetPasswordFor` will force the creation of a new Keychain, you can comment out/ignore `$jamf_binary resetPassword -updateLoginKeychain -username $resetUser -oldPassword $oldPass -password $newPass`
