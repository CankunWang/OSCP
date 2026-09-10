# OSCP 报告与截图规范
> [!DANGER] 核心原则
> 报告要让评审**不看靶机也能完整复现攻击**。每一处关键操作都要有：**命令 + 输出截图 + 上下文说明**，证据链闭环。

## 报告结构
- [ ] **执行摘要（Executive Summary）**
	- [ ] 一段话总结项目范围、测试时间、总体结论（拿到了几台、域控制权与否）
	- [ ] 风险等级总览表（高危/中危/低危数量）
	- [ ] 用非技术语言说明业务影响，管理层可读
- [ ] **方法（Methodology）**
	- [ ] 说明测试阶段：信息收集 → 漏洞发现 → 利用 → 提权 → 后渗透/横向
	- [ ] 说明遵循的框架（如参照 OWASP / PTES），列出工具清单
- [ ] **发现（Findings）**
	- [ ] 每个发现独立成节，用统一格式（见下方模板）
	- [ ] 按严重程度排序，漏洞要有 CVE/参考链接
- [ ] **攻击链（Attack Narrative / Kill Chain）**
	- [ ] 按时间顺序把每台机器的完整路径串起来（初始访问 → 提权 → flag）
	- [ ] 关键节点给 step-by-step 复现命令
- [ ] **修复建议（Remediation）**
	- [ ] 每条漏洞对应一条可执行修复（补丁/配置/代码级），带优先级
	- [ ] 通用建议：补丁管理、最小权限、强口令、网络分段、日志监控

### 单条 Finding 模板
```markdown
## [严重程度] 漏洞名称 - <IP>
- **受影响主机/端口：** 10.x.x.x:80
- **CVSS / 严重程度：** High
- **漏洞描述：** 一句话说明问题与成因
- **复现步骤：**
  1. ...
  ```bash
  # 命令
  ```
- **证据截图：** ![[finding_x.png]]
- **影响：** 能造成什么（RCE/越权/数据泄露）
- **修复建议：** 具体做法（升级版本/修改配置/加输入校验）
```

## 截图规范（OSCP 必须）
- [ ] **flag 截图必须三要素同框**：`whoami`（或 `id`）、`ipconfig`（或 `ifconfig`/`ip a`）、`type flag.txt` / `cat flag.txt` 的**内容**在同一终端窗口
	```cmd
	whoami
	ipconfig
	type C:\Users\<user>\Desktop\local.txt
	```
	```bash
	id
	ifconfig
	cat /home/<user>/local.txt
	```
- [ ] flag 必须用 `type`/`cat` 从**原始位置**读取并显示内容，不接受 web shell 等其他方式拿 flag
- [ ] **独立机器**（standalone）截图：`local.txt` 与 `proof.txt` 分别截，包含三要素
- [ ] **域环境**：`local.txt`（每台机器）+ 域控 `proof.txt`，截图同样三要素同框
- [ ] 截图要清晰、完整、可读，不要裁掉关键输出；一张图包含完整命令上下文
- [ ] 截图命名规范：`local_<IP>.png`、`proof_<IP>.png`、`priv_esc_<IP>_<步骤>.png`

## 报告要求
- [ ] **每个漏洞/提权步骤都要有截图**，不能只有文字描述
- [ ] **命令可复现**：报告里的每条命令评审能直接抄下来跑出同样结果
- [ ] **证据链完整**：从扫描发现 → 利用 → 拿到 shell → 提权 → flag，中间每一步都有截图衔接
- [ ] 记录失败的尝试（简述即可），证明枚举做全了，不是碰运气
- [ ] 截图里的 IP、hostname、flag 内容与正文一致；避免 PII 泄露
- [ ] 报告导出 PDF，命名如 `OSCP-OS-XXXXX-Exam-Report.pdf`，附上 Lab 报告（如要求）

## 参考开源项目
- [ ] OSCP 官方报告样例与要求（Offensive Security Exam Guide，https://help.offsec.com/）
- [ ] whoisflynn OSCP 报告模板（https://github.com/whoisflynn/OSCP-Exam-Report-Template-Markdown）
- [ ] noraj/OSCP-Exam-Report-Template-Markdown（https://github.com/noraj/OSCP-Exam-Report-Template-Markdown）
