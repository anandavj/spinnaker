# Spinnaker Installation
## Install Halyard
1. Install Java `apt install defult-jre` and `apt install default-jdk`
2. Update and Upgrade `sudo apt-get update && sudo apt-get upgrade`
3. Add user `sudo adduser spinnaker`
4. `curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh`
5. `sudo bash InstallHalyard.sh`
6. Check if Halyard was installed properly `hal-v`
## Choose Cloud Provider
