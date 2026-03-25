---
cmd_type: PrivEsc
os: Linux
tags:
  - cmd
syntax: DISPLAY=dummy SSH_ASKPASS=/tmp/ap.sh SSH_ASKPASS_REQUIRE=force setsid ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no user@<IP> whoami < /dev/null
---

适合没有交互 TTY，或者需要脚本化测试口令时使用。

```bash
DISPLAY=dummy SSH_ASKPASS=/tmp/ap.sh SSH_ASKPASS_REQUIRE=force \
setsid ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no \
-o StrictHostKeyChecking=no user@<IP> whoami < /dev/null
```

多跳时可以直接用 ProxyCommand：

```bash
ssh -i ~/.ssh/pivot_key \
  -o ProxyCommand='ssh -i ~/.ssh/pivot_key -W %h:%p user1@<pivot_ip>' \
  user2@<target_ip> whoami
```

