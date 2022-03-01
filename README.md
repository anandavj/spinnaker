# Spinnaker Installation
## Install Halyard
1. Install Java `apt install defult-jre` and `apt install default-jdk`
2. Update and Upgrade `sudo apt-get update && sudo apt-get upgrade`
3. Add user `sudo adduser spinnaker`
4. `curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh`
5. `sudo bash InstallHalyard.sh`
6. Check if Halyard was installed properly `hal -v`
## Choose Cloud Provider (I'm Using EKS)
1. Download and Install Kubectl
	```
	curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
	chmod +x ./kubectl
	sudo mv ./kubectl /usr/local/bin/kubectl
	```
	Verify the installation `kubectl help`
	
2. Download and Install aws-iam-authenticator
	```
	curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.13.7/2019-06-11/bin/linux/amd64/aws-iam-authenticator
	chmod +x ./aws-iam-authenticator
	mkdir -p $HOME/bin && cp ./aws-iam-authenticator $HOME/bin/aws-iam-authenticator && export PATH=$HOME/bin:$PATH
	echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
	```
	Verify the installation `aws-iam-authenticator help`
