---
title: "用 Claude Code hook 擋危險指令"
date: 2026-11-02T11:00:00+08:00
lastmod: 2026-11-02T11:00:00+08:00
draft: true
description: "同樣是 PreToolUse hook，擋危險指令的要 fail-closed，提醒用的要 fail-open。用兩支 bash 腳本說明兩種設計、onFailure 設定，以及怎麼測試 hook。"
categories: [ai]
tags: [claude-code, bash]
---

在 CLAUDE.md 寫「不要 force push」，模型大多會遵守，但「大多」不夠。Claude Code 的 PreToolUse hook 會在每次工具呼叫之前執行你的腳本，由腳本決定放行或擋下，模型沒辦法用說理的方式繞過去。

寫 hook 時最容易忽略的問題是：**hook 自己出錯的時候該怎麼辦？** 擋危險指令的護欄，出錯時應該擋下（fail-closed）；只是提醒的 hook，出錯時應該安靜放行（fail-open）。這篇用兩支 bash 腳本說明這兩種設計。

<!--more-->

## PreToolUse hook 的基本規則

Hook 寫在 settings 檔裡，分成三層：事件、matcher、handler。下面這段放在專案的 `.claude/settings.json`，讓每次呼叫 Bash 工具之前都先執行兩支腳本：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/guard.sh",
            "args": [],
            "onFailure": "block"
          },
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/push-reminder.sh",
            "args": [],
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

`matcher` 只由英數字、`_`、`-`、空白、`,`、`|` 組成時是精確比對，`Bash` 就只對應 Bash 工具，`Edit|Write` 則對應其中任一個。含有其他字元時，會被當成 JavaScript 正規表示式，而且不會自動錨定頭尾。有 `args` 時是 exec 形式，不經過 shell，路徑裡的變數比較不容易出錯。

腳本從 stdin 收到一份 JSON，Bash 工具的指令在 `tool_input.command`：

```json
{
  "hook_event_name": "PreToolUse",
  "cwd": "/home/user/my-project",
  "tool_name": "Bash",
  "tool_input": { "command": "git push --force origin main" }
}
```

實際的 payload 還有 `session_id`、`permission_mode` 等欄位。在 subagent 裡觸發時，還會多出 `agent_id` 和 `agent_type`。

腳本用結束碼回答：

| 結束碼 | 效果 |
|---|---|
| `0` | 成功。stdout 如果是 JSON，會依內容決定；沒有 JSON 就放行 |
| `2` | 擋下這次工具呼叫，stderr 的內容會回傳給 Claude |
| 其他（包括 `1`） | 非阻擋錯誤：記錄一個 hook error，**工具照樣執行** |

最後一列是第一個陷阱。Unix 習慣用 `exit 1` 表示失敗，但對 PreToolUse 來說，`exit 1` 等於放行。要擋下就一定要 `exit 2`。

除了結束碼，也可以 `exit 0` 並在 stdout 輸出 JSON，用 `permissionDecision` 給出 `allow`、`deny`、`ask` 或 `defer`。`ask` 會跳出確認視窗問使用者，等一下的提醒 hook 就是用這個。

## 兩種失敗哲學

| | 護欄（guard） | 提醒（reminder） |
|---|---|---|
| 目的 | 擋下不可挽回的操作 | 在可能出錯的地方多問一句 |
| 自己出錯時 | 擋下（fail-closed） | 放行，什麼都不說（fail-open） |
| 比對方式 | 寬鬆，寧可誤擋 | 嚴謹，避免誤報 |
| 誤判的代價 | 誤擋：改寫指令就好 | 誤報：每次都被打擾 |

判斷的依據是「漏擋」和「誤擋」哪一個比較貴。force push 蓋掉別人的 commit、`curl | sh` 跑了來路不明的腳本，這些都救不回來；誤擋頂多讓 Claude 換個寫法。提醒則相反，它就算漏了也沒什麼損失，但如果動不動就誤報，或是一壞掉就卡住所有 Bash 指令，大家很快就會把它關掉。

## 護欄：fail-closed

```bash
#!/usr/bin/env bash
# PreToolUse(Bash) guard, fail-closed: when in doubt, block.
set -u

block() {
  echo "已被 guard 擋下：$1。若是誤擋（例如訊息文字裡剛好有這個字串），請改寫指令，不要嘗試繞過。" >&2
  exit 2
}

# 解析不了就擋下，而不是放行
cmd=$(jq -er '.tool_input.command' 2>/dev/null) || block "無法解析 hook 輸入"

if grep -qE '(curl|wget)[^|]*\|[[:space:]]*(sudo[[:space:]]+)?(ba|z)?sh' <<<"$cmd"; then
  block "不可以把下載的內容直接 pipe 給 shell 執行"
fi
if grep -qE 'git[^;&|]*(commit|push)[^;&|]*--no-verify' <<<"$cmd"; then
  block "不可以用 --no-verify 跳過 git hook"
fi
no_lease=${cmd//--force-with-lease/}
if grep -qE 'git[^;&|]*push[^;&|]*(--force|[[:space:]]-[a-zA-Z]*f[a-zA-Z]*)([[:space:];&|"]|$)' <<<"$no_lease"; then
  block "不可以 force push，請改用 --force-with-lease"
fi
exit 0
```

幾個刻意的設計：

- **`jq -e` 失敗就擋下**。輸入不是 JSON、沒有 `command` 欄位，甚至機器上沒裝 jq，都會走到 `block`。
- **用純字串比對**。`git commit -m "never use git push --force"` 也會被擋，這是已知的誤擋，而且是故意的。為了分辨引號內外而把解析寫得很複雜，等於製造更多出錯的機會；誤擋只要改寫指令就能解決。
- **擋下的訊息是寫給 Claude 看的**。stderr 會回傳給模型，所以要說清楚為什麼擋、該怎麼做，並明確告訴它不要嘗試繞過。
- **先去掉 `--force-with-lease` 再比對**，讓安全的寫法可以通過。

寫這篇文章時，我自己環境裡的 guard 就擋過我一次：我用 heredoc 建立測試腳本，內容剛好有 `curl … | bash` 這段測試字串。解法是改用其他方式寫檔，而不是把 guard 關掉。誤擋就是這樣處理。

### 腳本自己的 fail-closed 還不夠

上面的腳本只能處理「腳本有在跑」的情況。如果腳本根本沒啟動（路徑寫錯、忘了 `chmod +x`、`env` 找不到 bash），或是執行超過 timeout，結束碼就不是 0 或 2。依照官方文件，這種失敗在大多數事件上預設會**放行**。一支路徑寫錯的護欄，等於讓所有指令都通過。

所以護欄的設定要加上 `"onFailure": "block"`（Claude Code v2.1.295 以後才有）。加上之後，下面這幾種情況都會被當成失敗，效果和 `exit 2` 一樣，會擋下工具呼叫：

- 腳本無法啟動
- 結束碼不是 0 也不是 2
- 超過 timeout
- 輸出的 JSON 無法解析，或不符合格式

腳本內的 `block` 處理的是「輸入看不懂」，`onFailure` 處理的是「腳本本身壞了」。兩層都要有，才是真正的 fail-closed。

## 提醒：fail-open

```bash
#!/usr/bin/env bash
# PreToolUse(Bash) advisory hook, fail-open: any error means "say nothing".

cmd=$(jq -r '.tool_input.command // empty' 2>/dev/null) || exit 0
[ -n "$cmd" ] || exit 0

# 去掉引號內的文字，讓 `git commit -m "git push origin main"` 不算數
# shortcut: ignores escaped quotes and heredocs; good enough for a reminder, not for a guard.
unquoted=$(sed -E "s/'[^']*'//g; s/\"[^\"]*\"//g" <<<"$cmd")

if grep -qE '(^|[;&|[:space:]])git[[:space:]]+push[^;&|]*[[:space:]](main|master)([[:space:];&|]|$)' <<<"$unquoted"; then
  jq -n '{hookSpecificOutput: {
    hookEventName: "PreToolUse",
    permissionDecision: "ask",
    permissionDecisionReason: "這個指令會直接 push 到主分支，確定不走 PR 嗎？"
  }}'
fi
exit 0
```

和護欄比較：

- **每條出錯的路都是 `exit 0`**，而且什麼都不輸出。解析失敗、沒裝 jq、payload 是空的，都當作沒看到。
- **比對比較嚴謹**。先拿掉引號裡的文字，再要求 `git` 出現在指令開頭或 `;`、`&&` 之後。因為誤報的代價是每次都被打擾，值得多寫幾行。
- **用 JSON 的 `ask`，而不是 `exit 2`**。提醒的目的是讓人確認，不是替人做決定。
- **設定裡給短 timeout，不加 `onFailure`**。這支 hook 在每一次 Bash 呼叫都會跑，萬一它卡住，至少 5 秒後就會放行。

文件也提到，settings 和 plugin 裡的 hook 在 subagent 裡**同樣會觸發**。如果你寫的 hook 只想管主對話，或只想管某一種 subagent，可以檢查 payload 的 `agent_id` 和 `agent_type` 自己決定範圍。plugin 裡的 hook 更應該 fail-open，因為它會在你預料不到的地方執行。

## 多個 hook 的結果怎麼合併

同一個事件可以掛好幾支 hook。多支 PreToolUse hook 給出不同決定時，優先順序是 `deny` > `defer` > `ask` > `allow`。所以護欄說擋、提醒說問的時候，結果是擋下。另外，hook 回傳 `allow` 並不會蓋過 settings 裡的 deny 和 ask 規則，那些規則還是會照常檢查。

## 測試 hook

Hook 只是讀 stdin、寫 stdout 的程式，不需要開 Claude Code 就能測。我用一支 bash 腳本，把組好的 payload 餵給 hook，再檢查結束碼和輸出：

```bash
payload() { jq -cn --arg c "$1" '{hook_event_name:"PreToolUse", tool_name:"Bash", tool_input:{command:$c}}'; }

expect guard.sh 2 - "$(payload 'git push -fu origin feature')"                       "-fu bundle"
expect guard.sh 0 - "$(payload 'git push --force-with-lease origin feature')"        "lease allowed"
expect guard.sh 2 - 'not json'                                                       "malformed JSON blocks"
expect push-reminder.sh 0 '"ask"' "$(payload 'npm test && git push upstream master')" "chained push asks"
expect push-reminder.sh 0 - "$(payload 'git commit -m "git push origin main later"')"  "quoted text silent"
expect push-reminder.sh 0 - 'not json'                                               "malformed JSON allows silently"
```

總共 21 個案例，在 macOS 的 bash 3.2 上全部通過，包括：

- 護欄擋下 `curl | bash`、`wget | sudo sh`、`--no-verify`、`--force`、`-fu`
- `--force-with-lease`、commit message 裡提到 `git push`、分支名稱含有 `f` 的 `git push -u` 都會放行
- 已知的誤擋（訊息裡寫 `git push --force`）確實會被擋，測試把這件事明確寫下來
- 輸入不是 JSON、沒有 `command`、沒裝 jq 時，護欄擋下，提醒安靜放行
- 用 `env` 啟動卻找不到 bash 時，結束碼是 127，這正是需要 `onFailure: "block"` 的情況

第一次跑測試就抓到兩個 bug：`-fu` 這種組合旗標沒被擋到，訊息結尾的 `--force"` 也因為後面接著引號而沒有比對成功。Hook 的比對規則很容易寫錯，一定要有測試。

## 限制

- **字串比對擋得住失誤，擋不住刻意繞過**。把指令藏在變數、`eval` 或 alias 裡就能繞過。護欄的對象是「不小心」的模型，不是攻擊者。
- **只管得到 Claude 的工具呼叫**。使用者自己在終端機下的指令不會經過 hook。
- **`onFailure` 需要 Claude Code v2.1.295 以後的版本**，舊版的護欄壞掉時會放行。
- **Windows**：shell 形式的 hook 在 Windows 上用 Git Bash 或 PowerShell 執行，行為和這篇的 bash 範例不同，要另外測試。

## 小結

| | 護欄 | 提醒 |
|---|---|---|
| 擋下的方式 | `exit 2` ＋ stderr 說明 | `exit 0` ＋ JSON `ask` |
| 輸入看不懂 | `exit 2` | `exit 0`，不輸出 |
| 腳本本身壞掉 | `"onFailure": "block"` | 不設定，加短 timeout |
| 比對 | 寬鬆，接受誤擋 | 嚴謹，避免誤報 |

寫 hook 前先問一句：這支 hook 壞掉時，我希望所有指令都停下來，還是都照常執行？答案決定了它該 fail-closed 還是 fail-open，後面的寫法都從這裡推出來。
