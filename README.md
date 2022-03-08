# Spinnaker Installation
## Install Halyard
1. Install Java `apt install default-jre -y && apt install default-jdk -y`
2. Update and Upgrade `sudo apt-get update && sudo apt-get upgrade`
3. Add user `sudo adduser halyard`
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

3. Install awscli `sudo apt install python-pip awscli` then configure aws by `aws configure`. **Don't forget to enable [these policies](https://github.com/anandavj/spinnaker/blob/main/aws-policy.json) if you use non-root aws user before continuing the step**
4. Install eksctl
	```
	curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/0.85.0/eksctl_Linux_amd64.tar.gz" | tar xz -C /tmp
	sudo mv /tmp/eksctl /usr/local/bin
	```
	Verify the installation `eksctl help`

5. Create an Amazon EKS cluster for Spinnaker `eksctl create cluster --name=eks-spinnaker --tags spinnaker=test --nodes=2 --region=ap-southeast-1  --write-kubeconfig=false`
6. Retrieve Amazon EKS cluster kubectl contexts `aws eks update-kubeconfig --name eks-spinnaker --region ap-southeast-1 --alias eks-spinnaker`
7. Add and configure Kubernetes accounts 
	```
	# Enable the Kubernetes provider
	hal config provider kubernetes enable
	
	# Set the current kubectl context to the cluster for Spinnaker
	kubectl config use-context eks-spinnaker
	
	# Assign the Kubernetes context to CONTEXT
	CONTEXT=$(kubectl config current-context)
	```
8. Create Kube namespace `kubectl create namespace spinnaker`
9. Create [service-account.yml](https://github.com/anandavj/spinnaker/blob/main/service-account.yml)
10. create a service account for the Amazon EKS cluster `kubectl apply --context $CONTEXT -f service-account.yml`
11. Extract the secret token of the created spinnaker-service-account
	```
	TOKEN=$(kubectl get secret --context $CONTEXT \
	$(kubectl get serviceaccount spinnaker-service-account \
	--context $CONTEXT \
	-n spinnaker \
	-o jsonpath='{.secrets[0].name}') \
	-n spinnaker \
	-o jsonpath='{.data.token}' | base64 --decode)
	```
12. Set the user entry in kubeconfig
	```
	kubectl config set-credentials ${CONTEXT}-token-user --token $TOKEN
	kubectl config set-context $CONTEXT --user ${CONTEXT}-token-user
	```
13. Add eks-spinnaker cluster as a Kubernetes provider `hal config provider kubernetes account add eks-spinnaker --context $CONTEXT`
14. Enable artifact support `hal config features edit --artifacts true`
15. Configure Spinnaker to install in Kubernetes `hal config deploy edit --type distributed --account-name eks-spinnaker`
16. Configure Spinnaker to use AWS S3
	```
	export YOUR_ACCESS_KEY_ID=<access-key>
	hal config storage s3 edit --access-key-id $YOUR_ACCESS_KEY_ID \
	--secret-access-key --region ap-southeast-1
	```
17. set the storage source to S3 `hal config storage edit --type s3`
18. List Version `hal version list`
19. Configure Halyard Spinnaker Version
	```
	export VERSION=1.19.2
	hal config version edit --version $VERSION
	```
20. Give Root RWX Permission to halyard group
	```
	chgrp halyard .
	chmod g+rwx .
	```
21. Install Spinnaker on the eks-spinnaker Amazon EKS cluster `hal deploy apply`
23. Verify the installation `kubectl -n spinnaker get svc`
24. Expose Spinnaker using Elastic Load Balancer
	```
	export NAMESPACE=spinnaker
	# Expose Gate and Deck
	kubectl -n ${NAMESPACE} expose service spin-gate --type LoadBalancer \
	  --port 80 --target-port 8084 --name spin-gate-public

	kubectl -n ${NAMESPACE} expose service spin-deck --type LoadBalancer \
	  --port 80 --target-port 9000 --name spin-deck-public

	export API_URL=$(kubectl -n $NAMESPACE get svc spin-gate-public \
	 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

	export UI_URL=$(kubectl -n $NAMESPACE get svc spin-deck-public \
	 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

	# Configure the URL for Gate
	hal config security api edit --override-base-url http://${API_URL}

	# Configure the URL for Deck
	hal config security ui edit --override-base-url http://${UI_URL}

	# Apply your changes to Spinnaker
	hal deploy apply
	```
25. Get the Deck URL `kubectl -n $NAMESPACE get svc spin-deck-public -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'`

# Jenkins Installation (Different Droplet / Cloud)
1. Update package `sudo apt-get update -y && sudo apt-get upgrade`
2. Install JDK `sudo apt install default-jdk -y`
3. Get Jenkins Installation Package and Install
	```
	curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee   /usr/share/keyrings/jenkins-keyring.asc > /dev/null
	echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]   https://pkg.jenkins.io/debian-stable binary/ | sudo tee   /etc/apt/sources.list.d/jenkins.list > /dev/null
	sudo apt-get update
	sudo apt-get install jenkins
	```
4. Start Jenkins Service `sudo service jenkins start`
5. Go to the UI `<DropletURL>:8080`
6. get first-time Password `cat /var/lib/jenkins/secrets/initialAdminPassword`
7. Install Plugins

# Jenkin and Spinnaker Integration
1. Enable the Jenkin Master `hal config ci jenkins enable`
2. Add Jenkins Master named **my-jenkins-master**
	```
	export BASEURL=<Your Jenkin's URL>
	export USERNAME=<Your Jenkin's username>
	hal config ci jenkins master add my-jenkins-master \
	    --address $BASEURL \
	    --username $USERNAME \
	    --password
	```
3. Configure Halyard to enable the csrf flag `hal config ci jenkins master edit MASTER --csrf true`
4. Apply changes `hal deploy apply`
