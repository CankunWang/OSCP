---
syntax:
---
cd ~/bloodhound-ce
sudo docker compose up -d
然后如果重置过数据库，这个命令查看初始密码
sudo docker logs bh-app 2>&1 | grep -i "Initial Password"