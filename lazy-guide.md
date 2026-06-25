# This is a guide with just the commands to deploy Akvorado with no explaination. Read the akvorado-lab-guide first

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release

sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker akvorado

sudo systemctl enable docker
sudo systemctl start docker

docker run hello-world

mkdir ~/akvorado
cd ~/akvorado

curl -sL https://github.com/akvorado/akvorado/releases/latest/download/docker-compose-quickstart.tar.gz | tar zxvf -


docker compose up -d

visit
http://<YOUR-VM-IP>:8081

edit outlet.yaml and set snmp

---
metadata:
  providers:
    - type: snmp
      credentials:
        ::/0:
          communities: public

# add this if you are using mikrotik

 core:
   default-sampling-rate: 1
```
Need to ensure that the interface description on the router matches the regex in the outlet.yaml file. eg `transit : ISP`


---
metadata:
  providers:
    - type: snmp
      credentials:
        ::/0:
          user-name: Username
          authentication-protocol: SHA
          authentication-passphrase: "pass"
          privacy-protocol: AES
          privacy-passphrase: "pass"
          
