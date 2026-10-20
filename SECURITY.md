# SonarQube Installation
```bash 
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```
To access: publicIP:9000 (by default username & password is admin)
status: success

# Trivy Installation
1. 
```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy        
```
status: failed
2. 
```bash
docker run -v /var/run/docker.sock:/var/run/docker.sock -v $HOME/Library/Caches:/root/.cache/ aquasec/trivy:0.76.0 image python:3.4-alpine
```
status: failed
3. 
```bash
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sudo sh -s -- -b /usr/local/bin v0.76.0
```
status: success
