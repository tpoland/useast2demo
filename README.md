# Notes
This repo was used along with the AWS codex to stand-up resources for the Digital Globe training (November 2018). Steps for deploying each component have been added to the [AWS Codex Doc](https://github.com/starkandwayne/codex/blob/master/docs/aws.md). Beyond the notes below, it should be used as the source of truth for steps to stand things up.

### Challenges
#### Terraform
The [jumpbox init script](terraform/aws/aws.tf) is inline as part of terraform/aws/aws.tf.

The Terreform scripts use Ubuntu Bionic Beaver for standing up the Jumpbox. We found that Ubuntu 18.04 was doing some internal updates when the Jumpbox first came online which was interfering with our jumpbox setup scripts. To get these to run we added a `sleep 30` directive to the jumpbox init script.

We also found that Jumpbox Ruby was being installed before `zlib1g-dev` and `libssl-dev` resulting in errors when trying to deploy the BOSH director. ***Experimental:*** To address this we added an apt upgrade as part of the init script before `sudo jumpbox system` (this has not yet been tested, and perhaps jumpbox system should do this).

The BOSH cli version default on the jumpbox is 3.1 so this was upgraded manually to 5.1.2 as part of our prep, it might make sense to add this to the jumpbox system updates or to the jumpbox init script.

***Note:*** The code changes for all of the above are not currently checked in to the code repository since they are not *"clean"*. The diff:

```diff
--- a/terraform/aws/aws.tf
+++ b/terraform/aws/aws.tf
@@ -1943,6 +1943,17 @@ resource "aws_instance" "bastion" {
     inline = [
       "sudo curl -o /usr/local/bin/jumpbox https://raw.githubusercontent.com/starkandwayne/jumpbox/master/bin/jumpbox",
       "sudo chmod 0755 /usr/local/bin/jumpbox",
+        # -----
+        # Unknown OS updates are blocking jumpbox init, we'll need to work around this but for now just adding arbitrary sleep directive... "Process exited with status 100".
+        # -----
+        #"ps -ef >> /tmp/jbox_plist.txt 2>&1",
+        #"lsof >> /tmp/jbox_lsof.txt 2>&1",
+        #"cycle=0 && while test `ps -ef | grep '[s]eed.loaded' | wc -l` -gt 0; do if test $${cycle} -gt 60; then break; fi; echo \"Waiting for process snap seed load: $${cycle}\" >> /tmp/wait.txt 2>&1; sleep 0.5; cycle=$$((cycle + 1)); done",
+        #"cycle=0 && while test `ps -ef | grep '[u]buntu-release-upgrader' | wc -l` -gt 0; do if test $${cycle} -gt 60; then break; fi; echo \"Waiting for process ubuntu-release-upgrader: $${cycle}\" >> /tmp/wait.txt 2>&1; sleep 0.5; cycle=$$((cycle + 1)); done",
+        # -----
+      "sleep 30", # punt on the jumpbox init conflict for now
+      "sudo apt install -y zlib1g-dev libssl-dev",
+      # we should add something to upgrade the bosh cli?
       "sudo jumpbox system"
     ]
     connection {
```

#### Concourse
The concourse manifest and cloud_config sets concourse up in the DMZ instead of the internal network and removes the haproxy service. This is because we were unable to connect haproxy so it was routable and experimented with using an ELB before settling on attaching an Elastic IP directly to the Concourse webserver.

Ultimately this will need to be revisited, but for the purposes of this training the current configuration will serve the purpose.

### Setting Up Users
The main thing that students need for accessing the course is the creds.yml which includes the following:

- Bosh Director (director_ssl::
  - CA (ca:)
  - URL (bosh_url:)
  - username (username:)
  - password (admin_password:)
- Concourse (concourse:)
  - URL (url:)
  - Username (username:)
  - Password (password:)
  - CA (ssl_ca:)

This can be refreshed for each user with the following snippet:

```
for u in norm lbunt maxpower
do
  echo "Copying creds for: ${u}"
  sudo rsync -og --chown=${u}:staff /home/${USER}/creds.yml /home/${u}/
done
```

## Concourse setup notes

We are using gpg to encrypt the dg_class file to the file name dg_class.gpg

1) Do your fly login before running the gen_teams script.   Name the target 'training'
   1. login with main team admin account
   1. decrypt dg_class.gpg if you have not done already by running bin/decrypt
      the argument is the passphrase 
   1. run bin/gen_teams from the top level directory

2) team name the student's first name initial + lastname 
    concourse username is 'admin'
    concourse password can be found dg_class file

