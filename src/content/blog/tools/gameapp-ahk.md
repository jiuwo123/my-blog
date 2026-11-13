---
link: 'tools/gameapp-ahk'
title: GameApp-Ahk
date: 2026-11-07
description: 自己随意创造的小玩具
categories:
  - 工具
tags:
  - AHK
colophon:
  - outdated
---

<style>
/* 代码块：淡蓝背景 + 深蓝灰文字 */
pre {
  background-color: #EAF3FB !important;
  border: 1px solid #CFE2F3 !important;
  border-radius: 8px;
  padding: 12px 14px;
  line-height: 1.6;
}
pre code { background: transparent !important; color: #204060 !important; }
code { background-color: #EAF3FB; color: #204060; border-radius: 4px; padding: 1px 5px; }
</style>

# 酒窝宏管理工具 · 源码解析文档


## 默认键位一览

| 变量 | 功能 | 默认值 |
| --- | --- | --- |
| `DriftKey` | 漂移键 | `q` |
| `NitroKey` | 氮气键 | `e` |
| `BoostKey` | 小喷键 | `Up` |
| `ForwardKey` | 前进键 | `w` |
| `Hotkey_BaoLi` | 暴力出弯 | `PgDn` |
| `Hotkey_WuHen` | 无痕三喷 | `Ctrl` |
| `Hotkey_Cut` | 快速切断 | `Enter` |
| `Hotkey_KaQi1` | 一段连卡 | `Left` |
| `Hotkey_KaQi2` | 二段连卡 | `Down` |
| `Hotkey_LianKa` | 三四五连卡 | `Right` |
| `Hotkey_F1` | 总开关 | `F1` |
| `Hotkey_F2` | 隐藏窗口 | `F2` |
| `Hotkey_F3` | 重置 | `F3` |
| `Hotkey_F10` | 快速切断（单键启停） | `F10` |
| `Hotkey_F11` | 三四五连卡（单键启停） | `F11` |
| `Hotkey_F12` | 暴力出弯（单键启停） | `F12` |

## 默认延迟参数（ms）

| 参数 | 含义 | 默认值 |
| --- | --- | --- |
| `CutHoldTime` | 快速切断 · 按住 | 22 |
| `CutInterval` | 快速切断 · 间隔 | 22 |
| `KaQi1HoldTime` | 一段连卡 · 按住 | 22 |
| `KaQi1MinTotal` | 一段连卡 · 保底总时长 | 25 |
| `KaQi2FirstHold` | 二段连卡 · 一段按住 | 22 |
| `KaQi2WaitTime` | 二段连卡 · 两段之间等待 | 618 |
| `KaQi2SecondHold` | 二段连卡 · 二段按住 | 22 |
| `LianKaFirstHold` | 三四五连卡 · 首按 | 22 |
| `LianKaWaitTime` | 三四五连卡 · 间隔 | 618 |
| `LianKaSecondHold` | 三四五连卡 · 连按 | 22 |
| `LianKaCount` | 连卡数量（3/4/5） | 4 |
| `LianKaMode` | 连卡模式（1=A 先漂移 / 2=B 首段带小喷） | 1 |
| `BaoLiVersion` | 暴力版本（珊瑚 / 酒窝） | 珊瑚 |
| `BaoLiHoldTime` | 酒窝暴力 · 按住时长（50–500） | 300 |

---

## 一、头部指令与权限检查

脚本编译版本声明、全局选项、发送模式、延迟设置，以及管理员权限校验与高精度计时器频率获取。

```ahk
; <COMPILER: v1.1.37.02>
#NoEnv
#SingleInstance, Force
#Persistent
#InstallKeybdHook
#InstallMouseHook
#KeyHistory, 0
#MaxHotkeysPerInterval, 1000000
SendMode, Input
ListLines, Off
SetBatchLines, -1
SetKeyDelay, -1, -1
SetMouseDelay, -1
SetControlDelay, -1
SetWinDelay, -1
if not A_IsAdmin {
  MsgBox, 16, 权限错误, 请右键以管理员身份运行本脚本！`n否则无法正常使用功能。
  ExitApp
}
Global Frequency := 0
if (!DllCall("QueryPerformanceFrequency", "Int64*", Frequency)) {
  MsgBox, 48, Error, Failed to get high precision timer frequency!
  ExitApp
}
```

## 二、默认键位与运行状态变量

四大驾驶键位、六大功能热键、各功能启停状态、各段延迟参数、连卡数量/模式与暴力版本等默认值。

```ahk
DriftKey := "q"
NitroKey := "e"
BoostKey := "Up"
ForwardKey := "w"
Hotkey_BaoLi := "PgDn"
Hotkey_WuHen := "Ctrl"
Hotkey_Cut := "Enter"
Hotkey_KaQi1 := "Left"
Hotkey_KaQi2 := "Down"
Hotkey_LianKa := "Right"
Hotkey_F1 := "F1"
Hotkey_F2 := "F2"
Hotkey_F3 := "F3"
Hotkey_F10 := "F10"
Hotkey_F11 := "F11"
Hotkey_F12 := "F12"
baoLiOn := false
wuHenOn := false
spaceCut := false
kaQi1On := false
kaQi2On := false
lianKaOn := false
Global BaoLiRunning := false
Global LianKaRunning := false
CutHoldTime := 22
CutInterval := 22
KaQi1HoldTime := 22
KaQi1MinTotal := 25
KaQi2FirstHold := 22
KaQi2WaitTime := 618
KaQi2SecondHold := 22
LianKaFirstHold := 22
LianKaWaitTime := 618
LianKaSecondHold := 22
LianKaCount := 4
LianKaMode := 1
BaoLiVersion := "珊瑚"
BaoLiHoldTime := 300
guiShow := true
KeyMapWindowOpen := false
DelayWindowOpen := false
OverviewWindowOpen := false
FKeyWindowOpen := false
```

## 三、配置文件路径

INI 配置文件存放于脚本同目录。

```ahk
ConfigFile := A_ScriptDir . "\酒窝配置.ini"
```

## 四、按键合法性校验 IsKeyValid()

判断输入的按键名称是否为 AHK 可识别的合法键位，支持功能键、方向键、鼠标键、单字母/数字/符号等。

```ahk
IsKeyValid(KeyName) {
  if (KeyName = "")
    return false
  if (StrLen(KeyName) > 15)
    return false
  ValidKeys := ["F1","F2","F3","F4","F5","F6","F7","F8","F9","F10","F11","F12","F13","F14","F15","F16","F17","F18","F19","F20","F21","F22","F23","F24"
  ,"Space","Tab","Enter","Return","Escape","Esc","Backspace","BS","Delete","Del","Insert","Ins","Home","End","PgUp","PgDn","PageUp","PageDown"
  ,"Up","Down","Left","Right"
  ,"Ctrl","Control","Shift","Alt","LWin","RWin","LControl","LShift","LAlt","RControl","RShift","RAlt"
  ,"Numpad0","Numpad1","Numpad2","Numpad3","Numpad4","Numpad5","Numpad6","Numpad7","Numpad8","Numpad9"
  ,"NumpadDot","NumpadEnter","NumpadAdd","NumpadSub","NumpadMult","NumpadDiv","NumpadDel","NumpadIns"
  ,"LButton","RButton","MButton","XButton1","XButton2","WheelUp","WheelDown","WheelLeft","WheelRight"]
  for index, valid in ValidKeys {
    if (KeyName = valid) {
      return true
    }
    StringLower, KeyLower, KeyName
    StringLower, ValidLower, valid
    if (KeyLower = ValidLower) {
      return true
    }
  }
  if (RegExMatch(KeyName, "^[a-zA-Z]$"))
    return true
  if (RegExMatch(KeyName, "^[0-9]$"))
    return true
  if (RegExMatch(KeyName, "^[!@#$%^&*()_+\-=\[\]{};:'"",./<>?\\|`~]$"))
    return true
  if (StrLen(KeyName) > 2)
    return false
  try {
    vk := GetKeyVK(KeyName)
    if (vk != "")
      return true
  }
  return false
}
```

## 五、读取配置 LoadConfig()

从 INI 读取键位、热键、延迟参数、连卡数量/模式与暴力版本；对越界取值做默认值兜底。

```ahk
LoadConfig() {
  global
  if !FileExist(ConfigFile)
    return
  IniRead, DriftKey, %ConfigFile%, KeyMapping, DriftKey, %DriftKey%
  IniRead, NitroKey, %ConfigFile%, KeyMapping, NitroKey, %NitroKey%
  IniRead, BoostKey, %ConfigFile%, KeyMapping, BoostKey, %BoostKey%
  IniRead, ForwardKey, %ConfigFile%, KeyMapping, ForwardKey, %ForwardKey%
  IniRead, Hotkey_BaoLi, %ConfigFile%, Hotkeys, BaoLi, %Hotkey_BaoLi%
  IniRead, Hotkey_WuHen, %ConfigFile%, Hotkeys, WuHen, %Hotkey_WuHen%
  IniRead, Hotkey_Cut, %ConfigFile%, Hotkeys, Cut, %Hotkey_Cut%
  IniRead, Hotkey_KaQi1, %ConfigFile%, Hotkeys, KaQi1, %Hotkey_KaQi1%
  IniRead, Hotkey_KaQi2, %ConfigFile%, Hotkeys, KaQi2, %Hotkey_KaQi2%
  IniRead, Hotkey_LianKa, %ConfigFile%, Hotkeys, LianKa, %Hotkey_LianKa%
  IniRead, Hotkey_F10, %ConfigFile%, Hotkeys, F10, %Hotkey_F10%
  IniRead, Hotkey_F11, %ConfigFile%, Hotkeys, F11, %Hotkey_F11%
  IniRead, Hotkey_F12, %ConfigFile%, Hotkeys, F12, %Hotkey_F12%
  IniRead, CutHoldTime, %ConfigFile%, Delays, CutHold, %CutHoldTime%
  IniRead, CutInterval, %ConfigFile%, Delays, CutInterval, %CutInterval%
  IniRead, KaQi1HoldTime, %ConfigFile%, Delays, KaQi1Hold, %KaQi1HoldTime%
  IniRead, KaQi1MinTotal, %ConfigFile%, Delays, KaQi1Min, %KaQi1MinTotal%
  IniRead, KaQi2FirstHold, %ConfigFile%, Delays, KaQi2First, %KaQi2FirstHold%
  IniRead, KaQi2WaitTime, %ConfigFile%, Delays, KaQi2Wait, %KaQi2WaitTime%
  IniRead, KaQi2SecondHold, %ConfigFile%, Delays, KaQi2Second, %KaQi2SecondHold%
  IniRead, LianKaFirstHold, %ConfigFile%, Delays, LianKaFirst, %LianKaFirstHold%
  IniRead, LianKaWaitTime, %ConfigFile%, Delays, LianKaWait, %LianKaWaitTime%
  IniRead, LianKaSecondHold, %ConfigFile%, Delays, LianKaSecond, %LianKaSecondHold%
  IniRead, LianKaCount, %ConfigFile%, Delays, LianKaCount, %LianKaCount%
  if (LianKaCount < 3 || LianKaCount > 5)
    LianKaCount := 4
  IniRead, LianKaMode, %ConfigFile%, Delays, LianKaMode, %LianKaMode%
  if (LianKaMode != 1 && LianKaMode != 2)
    LianKaMode := 1
  IniRead, BaoLiVersion, %ConfigFile%, Settings, BaoLiVersion, %BaoLiVersion%
  IniRead, BaoLiHoldTime, %ConfigFile%, Settings, BaoLiHoldTime, %BaoLiHoldTime%
}
```

## 六、保存配置 SaveConfig()

把当前键位、热键、延迟与设置写回 INI 文件。

```ahk
SaveConfig() {
  global
  IniWrite, %DriftKey%, %ConfigFile%, KeyMapping, DriftKey
  IniWrite, %NitroKey%, %ConfigFile%, KeyMapping, NitroKey
  IniWrite, %BoostKey%, %ConfigFile%, KeyMapping, BoostKey
  IniWrite, %ForwardKey%, %ConfigFile%, KeyMapping, ForwardKey
  IniWrite, %Hotkey_BaoLi%, %ConfigFile%, Hotkeys, BaoLi
  IniWrite, %Hotkey_WuHen%, %ConfigFile%, Hotkeys, WuHen
  IniWrite, %Hotkey_Cut%, %ConfigFile%, Hotkeys, Cut
  IniWrite, %Hotkey_KaQi1%, %ConfigFile%, Hotkeys, KaQi1
  IniWrite, %Hotkey_KaQi2%, %ConfigFile%, Hotkeys, KaQi2
  IniWrite, %Hotkey_LianKa%, %ConfigFile%, Hotkeys, LianKa
  IniWrite, %Hotkey_F10%, %ConfigFile%, Hotkeys, F10
  IniWrite, %Hotkey_F11%, %ConfigFile%, Hotkeys, F11
  IniWrite, %Hotkey_F12%, %ConfigFile%, Hotkeys, F12
  IniWrite, %CutHoldTime%, %ConfigFile%, Delays, CutHold
  IniWrite, %CutInterval%, %ConfigFile%, Delays, CutInterval
  IniWrite, %KaQi1HoldTime%, %ConfigFile%, Delays, KaQi1Hold
  IniWrite, %KaQi1MinTotal%, %ConfigFile%, Delays, KaQi1Min
  IniWrite, %KaQi2FirstHold%, %ConfigFile%, Delays, KaQi2First
  IniWrite, %KaQi2WaitTime%, %ConfigFile%, Delays, KaQi2Wait
  IniWrite, %KaQi2SecondHold%, %ConfigFile%, Delays, KaQi2Second
  IniWrite, %LianKaFirstHold%, %ConfigFile%, Delays, LianKaFirst
  IniWrite, %LianKaWaitTime%, %ConfigFile%, Delays, LianKaWait
  IniWrite, %LianKaSecondHold%, %ConfigFile%, Delays, LianKaSecond
  IniWrite, %LianKaCount%, %ConfigFile%, Delays, LianKaCount
  IniWrite, %LianKaMode%, %ConfigFile%, Delays, LianKaMode
  IniWrite, %BaoLiVersion%, %ConfigFile%, Settings, BaoLiVersion
  IniWrite, %BaoLiHoldTime%, %ConfigFile%, Settings, BaoLiHoldTime
}
```

## 七、刷新总览界面 RefreshOverview()

把当前键位、延迟、连卡与暴力版本同步刷新到“配置总览”窗口。

```ahk
RefreshOverview() {
  global
  GuiControl, Overview:, OV_Drift, %DriftKey%
  GuiControl, Overview:, OV_Nitro, %NitroKey%
  GuiControl, Overview:, OV_Boost, %BoostKey%
  GuiControl, Overview:, OV_Forward, %ForwardKey%
  GuiControl, Overview:, OV_Cut, 按住 %CutHoldTime%ms / 间隔 %CutInterval%ms
  GuiControl, Overview:, OV_K1, 按住 %KaQi1HoldTime%ms / 保底 %KaQi1MinTotal%ms
  GuiControl, Overview:, OV_K2, 一段 %KaQi2FirstHold%ms / 等待 %KaQi2WaitTime%ms / 二段 %KaQi2SecondHold%ms
  LianKaCountDesc := (LianKaCount = 3 ? "三连" : LianKaCount = 4 ? "四连" : "五连")
  LianKaModeDesc := (LianKaMode = 2 ? "模式B（首段带小喷）" : "模式A（先漂移）")
  GuiControl, Overview:, OV_LianKa, 首按 %LianKaFirstHold%ms / 间隔 %LianKaWaitTime%ms / 连按 %LianKaSecondHold%ms
  GuiControl, Overview:, OV_LianKaCount, 当前连卡：%LianKaCountDesc%（%LianKaModeDesc%）
  OV_VersionText := (BaoLiVersion = "珊瑚" ? "珊瑚暴力（固定300ms）" : "酒窝暴力（可调50-500ms）")
  GuiControl, Overview:, OV_Version, %OV_VersionText%
  GuiControl, Overview:, OV_HoldTime, %BaoLiHoldTime%ms
  Tooltip, 已刷新
  SetTimer, RemoveTooltip, -1000
}
```

## 八、同步所有界面 SyncAllGUI()

将当前变量同步到主窗口、键位映射、延迟参数、总览、F 功能键等已打开的子窗口控件。

```ahk
SyncAllGUI() {
  global
  GuiControl, Main:, BaoLiEdit, %Hotkey_BaoLi%
  GuiControl, Main:, WuHenEdit, %Hotkey_WuHen%
  GuiControl, Main:, SpaceEdit, %Hotkey_Cut%
  GuiControl, Main:, KaQi1Edit, %Hotkey_KaQi1%
  GuiControl, Main:, KaQi2Edit, %Hotkey_KaQi2%
  GuiControl, Main:, LianKaEdit, %Hotkey_LianKa%
  if (KeyMapWindowOpen) {
    GuiControl, KeyMap:, KMP_Drift, %DriftKey%
    GuiControl, KeyMap:, KMP_Nitro, %NitroKey%
    GuiControl, KeyMap:, KMP_Boost, %BoostKey%
    GuiControl, KeyMap:, KMP_Forward, %ForwardKey%
  }
  if (DelayWindowOpen) {
    GuiControl, Delay:, DCutHoldEdit, %CutHoldTime%
    GuiControl, Delay:, DCutIntervalEdit, %CutInterval%
    GuiControl, Delay:, DK1HoldEdit, %KaQi1HoldTime%
    GuiControl, Delay:, DK1MinEdit, %KaQi1MinTotal%
    GuiControl, Delay:, DK2FirstEdit, %KaQi2FirstHold%
    GuiControl, Delay:, DK2WaitEdit, %KaQi2WaitTime%
    GuiControl, Delay:, DK2SecondEdit, %KaQi2SecondHold%
    GuiControl, Delay:, DLKFirstEdit, %LianKaFirstHold%
    GuiControl, Delay:, DLKWaitEdit, %LianKaWaitTime%
    GuiControl, Delay:, DLKSecondEdit, %LianKaSecondHold%
    if (LianKaCount = 3) {
      GuiControl, Delay:, DLK3, 1
    } else if (LianKaCount = 4) {
      GuiControl, Delay:, DLK4, 1
    } else {
      GuiControl, Delay:, DLK5, 1
    }
    if (LianKaMode = 2)
      GuiControl, Delay:, DLianKaModeCB, 1
    else
      GuiControl, Delay:, DLianKaModeCB, 0
    GuiControl, Delay:, DBaoLiHoldEdit, %BaoLiHoldTime%
    if (BaoLiVersion = "珊瑚") {
      GuiControl, Delay:, DBVShanhu, 1
      GuiControl, Delay:Disable, DBaoLiHoldEdit
    } else {
      GuiControl, Delay:, DBVJiuwo, 1
      GuiControl, Delay:Enable, DBaoLiHoldEdit
    }
  }
  if (FKeyWindowOpen) {
    GuiControl, FKey:, F10Edit, %Hotkey_F10%
    GuiControl, FKey:, F11Edit, %Hotkey_F11%
    GuiControl, FKey:, F12Edit, %Hotkey_F12%
  }
  UpdateBaoLiVersionUI()
  if (OverviewWindowOpen) {
    RefreshOverview()
  }
}
```

## 九、主窗口 GUI 构建

创建主界面：六大功能行、控制热键行、功能按钮区、状态提示与退出按钮，并设置标题、配色与字号。

```ahk
Gui, Main:New, +AlwaysOnTop -MinimizeBox -MaximizeBox +LastFound
Gui, Main:Color, F0F5FA, FFFFFF
Gui, Main:Font, s11 c222222, 微软雅黑
Gui, Main:Margin, 15, 15
Gui, Main:Font, s14 c0055BB, 楷体
Gui, Main:Add, Text, x0 y0 w450 h35 c0055BB, 作者：酒窝
Gui, Main:Add, Text, x0 y28 w450 h25 c0055BB, 联系方式：神龙见首不见尾
Gui, Main:Font, s11 c222222, 微软雅黑
Gui, Main:Add, Text, x480 y4 w110 h25 c666666, F2隐藏窗口
Gui, Main:Add, GroupBox, x0 y60 w600 h280 c3377CC, 六大功能
Gui, Main:Add, Text, x20 y90 w100 h25, 暴力出弯：
Gui, Main:Add, Edit, x130 y87 w100 h25 vBaoLiEdit, %Hotkey_BaoLi%
Gui, Main:Add, Button, x240 y87 w90 h25 gBindBaoLi, 绑定
Gui, Main:Add, Button, x340 y87 w100 h25 gToggleBaoLi, 启停
Gui, Main:Add, Text, x450 y90 w80 h25 vStateB cFF0000, 未启用
Gui, Main:Add, Text, x20 y130 w100 h25, 无痕三喷：
Gui, Main:Add, Edit, x130 y127 w100 h25 vWuHenEdit, %Hotkey_WuHen%
Gui, Main:Add, Button, x240 y127 w90 h25 gBindWuHen, 绑定
Gui, Main:Add, Button, x340 y127 w100 h25 gToggleWuHen, 启停
Gui, Main:Add, Text, x450 y130 w80 h25 vStateW cFF0000, 未启用
Gui, Main:Add, Text, x20 y170 w100 h25, 快速切断：
Gui, Main:Add, Edit, x130 y167 w100 h25 vSpaceEdit, %Hotkey_Cut%
Gui, Main:Add, Button, x240 y167 w90 h25 gBindSpace, 绑定
Gui, Main:Add, Button, x340 y167 w100 h25 gToggleSpace, 启停
Gui, Main:Add, Text, x450 y170 w80 h25 vStateS cFF0000, 未启用
Gui, Main:Add, Progress, x15 y205 w570 h2 cCCCCCC BackgroundCCCCCC, 100
Gui, Main:Add, Text, x20 y225 w100 h25, 一段连卡：
Gui, Main:Add, Edit, x130 y222 w100 h25 vKaQi1Edit, %Hotkey_KaQi1%
Gui, Main:Add, Button, x240 y222 w90 h25 gBindKaQi1, 绑定
Gui, Main:Add, Button, x340 y222 w100 h25 gToggleKaQi1, 启停
Gui, Main:Add, Text, x450 y225 w80 h25 vStateK1 cFF0000, 未启用
Gui, Main:Add, Text, x20 y265 w100 h25, 二段连卡：
Gui, Main:Add, Edit, x130 y262 w100 h25 vKaQi2Edit, %Hotkey_KaQi2%
Gui, Main:Add, Button, x240 y262 w90 h25 gBindKaQi2, 绑定
Gui, Main:Add, Button, x340 y262 w100 h25 gToggleKaQi2, 启停
Gui, Main:Add, Text, x450 y265 w80 h25 vStateK2 cFF0000, 未启用
Gui, Main:Add, Text, x20 y305 w100 h25, 三四五连卡：
Gui, Main:Add, Edit, x130 y302 w100 h25 vLianKaEdit, %Hotkey_LianKa%
Gui, Main:Add, Button, x240 y302 w90 h25 gBindLianKa, 绑定
Gui, Main:Add, Button, x340 y302 w100 h25 gToggleLianKa, 启停
Gui, Main:Add, Text, x450 y305 w80 h25 vStateLK cFF0000, 未启用
Gui, Main:Add, GroupBox, x0 y350 w600 h85 c3377CC, 控制热键（可自定义）
Gui, Main:Add, Text, x20 y375 w90 h20, F1（总开关）：
Gui, Main:Add, Edit, x120 y372 w90 h24 vF1Edit, %Hotkey_F1%
Gui, Main:Add, Button, x220 y372 w70 h24 gBindF1, 绑定
Gui, Main:Add, Text, x320 y375 w90 h20, F2（隐藏）：
Gui, Main:Add, Edit, x420 y372 w90 h24 vF2Edit, %Hotkey_F2%
Gui, Main:Add, Button, x520 y372 w70 h24 gBindF2, 绑定
Gui, Main:Add, Text, x20 y410 w90 h20, F3（重置）：
Gui, Main:Add, Edit, x120 y407 w90 h24 vF3Edit, %Hotkey_F3%
Gui, Main:Add, Button, x220 y407 w70 h24 gBindF3, 绑定
Gui, Main:Add, Button, x15 y455 w95 h28 gOpenKeyMappingWindow, 键位映射
Gui, Main:Add, Button, x115 y455 w95 h28 gOpenDelaySettingsWindow, 延迟参数
Gui, Main:Add, Button, x215 y455 w95 h28 gOpenOverviewWindow, 配置总览
Gui, Main:Add, Button, x315 y455 w95 h28 gOpenHelpWindow, 按键说明
Gui, Main:Add, Button, x415 y455 w85 h28 gSaveConfig, 保存配置
Gui, Main:Add, Button, x505 y455 w95 h28 gOpenFKeyWindow, F功能键
Gui, Main:Add, Text, x15 y488 w500 h20 vBindStatus c666666, 点击"绑定"修改键位
Gui, Main:Add, Text, x15 y510 w250 h20 cFF0000 vWindowStatus, 请切换到游戏窗口 (GameApp.exe)
Gui, Main:Add, Button, x0 y540 w600 h30 gExitApp cEE4444, 退出程序
Gui, Main:Font, s10 c0066CC, 楷体
Gui, Main:Add, Text, x0 y574 w600 vAuthorText c0066CC, 作者：酒窝 | 联系方式：就不告诉你
Gui, Main:Show, xCenter yCenter w620 h615, 宏管理工具
```

## 十、状态初始化与全局热键注册

初始化各功能状态颜色，载入并同步配置，注册 F1/F2/F3/F10/F11/F12 及各功能热键、方向键与数字切换键，启动窗口检测定时器。

```ahk
SetStateColor("StateB", "未启用", "FF0000")
SetStateColor("StateW", "未启用", "FF0000")
SetStateColor("StateS", "未启用", "FF0000")
SetStateColor("StateK1", "未启用", "FF0000")
SetStateColor("StateK2", "未启用", "FF0000")
SetStateColor("StateLK", "未启用", "FF0000")
LoadConfig()
SyncAllGUI()
Hotkey, $%Hotkey_F1%, F1Action, On
Hotkey, $%Hotkey_F2%, F2Action, On
Hotkey, $%Hotkey_F3%, F3Action, On
Hotkey, $%Hotkey_F10%, F10Action, On
Hotkey, $%Hotkey_F11%, F11Action, On
Hotkey, $%Hotkey_F12%, F12Action, On
Hotkey, $%Hotkey_BaoLi%, FuncWrapper, On
Hotkey, $%Hotkey_WuHen%, FuncWrapper, On
Hotkey, %Hotkey_Cut%, FuncWrapper, On
Hotkey, $%Hotkey_KaQi1%, FuncWrapper, On
Hotkey, $%Hotkey_KaQi2%, FuncWrapper, On
Hotkey, $%Hotkey_LianKa%, FuncWrapper, On
Hotkey, *Left, FuncWrapper, On
Hotkey, *Down, FuncWrapper, On
Hotkey, *Right, FuncWrapper, On
Hotkey, $3, FuncWrapper, On
Hotkey, $4, FuncWrapper, On
Hotkey, $5, FuncWrapper, On
SetTimer, CheckWindow, 500
OnMessage(0x201, "WM_LBUTTONDOWN")
```

## 十一、无边框拖动 WM_LBUTTONDOWN

主窗口标题区按下左键时发送拖动消息，实现无标题栏拖动。

```ahk
WM_LBUTTONDOWN() {
  if (!A_GuiControl)
    PostMessage, 0xA1, 2,,, A
}
```

## 十二、暴力版本 UI 同步 UpdateBaoLiVersionUI()

根据当前暴力版本（珊瑚/酒窝）切换延迟窗口输入框的可用状态，并刷新总览与主窗口提示。

```ahk
UpdateBaoLiVersionUI() {
  global BaoLiVersion, DelayWindowOpen
  if (DelayWindowOpen) {
    if (BaoLiVersion = "珊瑚") {
      GuiControl, Delay:, DBVShanhu, 1
      GuiControl, Delay:Disable, DBaoLiHoldEdit
    } else {
      GuiControl, Delay:, DBVJiuwo, 1
      GuiControl, Delay:Enable, DBaoLiHoldEdit
    }
  }
  if (OverviewWindowOpen)
    RefreshOverview()
  if (BaoLiVersion = "珊瑚") {
    GuiControl, Main:, BindStatus, 已切换到：珊瑚暴力（固定）| 按住时长固定300ms
  } else {
    GuiControl, Main:, BindStatus, 已切换到：酒窝暴力（可调）| 按住时长可调（50-500ms）
  }
}
```

## 十三、状态文本与颜色工具

SetStateColor / SetFKeyState 统一更新控件文字与颜色；UpdateFKeyStates 同步 F 功能键窗口的启用状态。

```ahk
SetStateColor(ControlName, Text, Color) {
  GuiControl, Main:, %ControlName%, %Text%
  GuiControl, Main:+c%Color%, %ControlName%
  GuiControl, Main:+Redraw, %ControlName%
}
SetFKeyState(ControlName, Text, Color) {
  global FKeyWindowOpen
  if (FKeyWindowOpen) {
    GuiControl, FKey:, %ControlName%, %Text%
    GuiControl, FKey:+c%Color%, %ControlName%
    GuiControl, FKey:+Redraw, %ControlName%
  }
}
UpdateFKeyStates() {
  global
  if (spaceCut) {
    SetFKeyState("StateF10", "已启用", "009933")
  } else {
    SetFKeyState("StateF10", "未启用", "FF0000")
  }
  if (lianKaOn) {
    SetFKeyState("StateF11", "已启用", "009933")
  } else {
    SetFKeyState("StateF11", "未启用", "FF0000")
  }
  if (baoLiOn) {
    SetFKeyState("StateF12", "已启用", "009933")
  } else {
    SetFKeyState("StateF12", "未启用", "FF0000")
  }
}
```

## 十四、高精度等待 PreciseSleep()

基于 QueryPerformanceCounter 的高精度延时：超过 15ms 先 Sleep 近似值再忙等校准，其余直接忙等。

:::encrypted{password='Game'}
PreciseSleep(Milliseconds) {
  Static AllowSleep := (Milliseconds > 15)
  targetMs := Milliseconds
  DllCall("QueryPerformanceCounter", "Int64*", startCounter)
  if (AllowSleep && targetMs > 15) {
    Sleep, % targetMs - 10 > 0 ? targetMs - 10 : 0
  }
  Loop {
    DllCall("QueryPerformanceCounter", "Int64*", currentCounter)
    elapsed := ((currentCounter - startCounter) * 1000) / Frequency
    if (elapsed >= targetMs)
      break
  }
}
:::

## 十五、清除悬浮提示 RemoveTooltip

定时器回调，清空 Tooltip 悬浮提示。

```ahk
RemoveTooltip:
Tooltip
return
```

## 十六、按键说明窗口 OpenHelpWindow

把内置帮助文本写入临时文件并用记事本打开；包含键位写法、各功能用法与版本说明。

```ahk
OpenHelpWindow:
HelpContent =
(
很重要必须要看！！！！！！
	右键管理员运行！！！右键管理员运行！！！！右键管理员运行！！！！！！

1.修改按键请务必按照写法
    例如：小键盘数字8修改小喷键为Numpad8（N必须大写）
          如果要修改的键位为功能键位，例如Enter，Tab，Ctrl，F1-F2等，逐个字母输入且首字母大写
          比如切断改为Tab，就依次输入T，a，b；回车键位就输入E，n，t,e,r
本软件没有修改按键映射没有写捕获功能，请务必好好输入，氮气键位游戏内是Ctrl和E，如果你是常用
e放氮气那你改为Ctrl（其实不改也一样）

2.二段连卡为q+（小喷+q），即卡接，要记住第一个是没有w键位的，需要定制的找酒窝（你找了我也不理你）
为什么卡接设置成q+w（w是小喷）q；因为有时候落地卡接会出现小喷的声音，用过森涩卡气宏应该知道

3.暴力出弯；拉三喷，小喷灯亮了可以按一下暴力出弯键，只要亮了就包出百分百

4.无痕：这个要压喷，压住按氮气会出；目前市面上的无痕宏都是这样

5.这点是最重要的，酒窝永远最帅！！！！

6.修改按键映射，请务必好好看看第一条说明；

7.延迟配置里面，默认珊瑚暴力就好了；切断和卡气的延迟我也调好了可以不用懂，拿到手改一下按键映射
就能直接用；如果你感兴趣可以改改延迟，60帧率大概一帧是16.67ms

8.qq号：3192821508，没事请勿加谢谢

9.2026-8.15,好了我新增了读取文件，改完按键保存配置就好了，同一台电脑不用每次打开都改按键了

10.市面上还有小喷宏和起步宏；但是他们是月卡起卖，太贵了，我没办法拿到借鉴，如果谁有办法做一个并想融合
到这里面请添加qq备注（我应该会感谢你，可能我也用的到？）

11.其实还有左侧卡宏，但我觉得大家一般大差不差我就没加进去，有需要的话再找我吧~

12.另，本软件无加壳（其实我想加但是感觉没必要），哈哈哈哈

13.新增三四五连卡（默认热键Right，可在界面改绑）：模式A：先按一下q起漂，之后每一次都是q和小喷同时按下；模式B（勾选"首段带小喷"）：第一下就带小喷，全程q+小喷同时按；`n连数（三连/四连/五连）、模式（A/B）和延迟（首按/间隔/连按）都在"延迟参数"窗口里调；游戏内可直接按 3/4/5 切换连卡数量（按3=三连 按4=四连 按5=五连，桌面按 3/4/5 正常输入数字，触发仍用Right）

)
FileDelete, %A_Temp%\酒窝按键说明.txt
FileAppend, %HelpContent%, %A_Temp%\酒窝按键说明.txt
Run, notepad.exe "%A_Temp%\酒窝按键说明.txt"
return
```

## 十七、延迟参数应用回调

快速切断、一段连卡、二段连卡、三四五连卡的参数从延迟窗口读回，更新状态提示与悬浮说明。

```ahk
ApplyCutDelayFromWindow:
Gui, Delay:Submit, NoHide
CutHoldTime := DCutHoldEdit + 0
CutInterval := DCutIntervalEdit + 0
GuiControl, Main:, BindStatus, 快速切断：按住 %CutHoldTime%ms / 间隔 %CutInterval%ms
Tooltip, 快速切断参数已更新
SetTimer, RemoveTooltip, -1500
return
ApplyK1DelayFromWindow:
Gui, Delay:Submit, NoHide
KaQi1HoldTime := DK1HoldEdit + 0
KaQi1MinTotal := DK1MinEdit + 0
GuiControl, Main:, BindStatus, 一段连卡：按住 %KaQi1HoldTime%ms / 保底 %KaQi1MinTotal%ms
Tooltip, 一段连卡参数已更新
SetTimer, RemoveTooltip, -1500
return
ApplyK2DelayFromWindow:
Gui, Delay:Submit, NoHide
KaQi2FirstHold := DK2FirstEdit + 0
KaQi2WaitTime := DK2WaitEdit + 0
KaQi2SecondHold := DK2SecondEdit + 0
GuiControl, Main:, BindStatus, 二段连卡：一段 %KaQi2FirstHold%ms / 等待 %KaQi2WaitTime%ms / 二段 %KaQi2SecondHold%ms
Tooltip, 二段连卡参数已更新
SetTimer, RemoveTooltip, -1500
return
ApplyLianKaDelayFromWindow:
Gui, Delay:Submit, NoHide
LianKaFirstHold := DLKFirstEdit + 0
LianKaWaitTime := DLKWaitEdit + 0
LianKaSecondHold := DLKSecondEdit + 0
LianKaCountDesc := (LianKaCount = 3 ? "三连" : LianKaCount = 4 ? "四连" : "五连")
GuiControl, Main:, BindStatus, 三四五连卡：首按 %LianKaFirstHold%ms / 间隔 %LianKaWaitTime%ms / 连按 %LianKaSecondHold%ms / %LianKaCountDesc%
Tooltip, 三四五连卡参数已更新
SetTimer, RemoveTooltip, -1500
return
```

## 十八、连卡数量 / 模式 / 暴力版本切换回调

单选/勾选变化时更新连卡数量、连卡模式（A/B）、暴力按住时长与暴力版本（珊瑚/酒窝），并刷新界面与提示。

```ahk
OnLianKaCountChange:
GuiControlGet, IsDLK3,, DLK3
GuiControlGet, IsDLK4,, DLK4
if (IsDLK3 = 1) {
  LianKaCount := 3
} else if (IsDLK4 = 1) {
  LianKaCount := 4
} else {
  LianKaCount := 5
}
LianKaCountDesc := (LianKaCount = 3 ? "三连" : LianKaCount = 4 ? "四连" : "五连")
GuiControl, Main:, BindStatus, 三四五连卡：首按 %LianKaFirstHold%ms / 间隔 %LianKaWaitTime%ms / 连按 %LianKaSecondHold%ms / %LianKaCountDesc%
if (OverviewWindowOpen)
  RefreshOverview()
Tooltip, 连卡已切换为：%LianKaCountDesc%
SetTimer, RemoveTooltip, -1500
return
OnLianKaModeChange:
GuiControlGet, IsModeB,, DLianKaModeCB
if (IsModeB = 1)
  LianKaMode := 2
else
  LianKaMode := 1
LianKaModeDesc := (LianKaMode = 2 ? "模式B（首段带小喷）" : "模式A（先漂移）")
GuiControl, Main:, BindStatus, 三四五连卡：%LianKaModeDesc%
if (OverviewWindowOpen)
  RefreshOverview()
Tooltip, 连卡模式已切换为：%LianKaModeDesc%
SetTimer, RemoveTooltip, -1500
return
ApplyBaoLiHoldFromWindow:
if (BaoLiVersion = "珊瑚") {
  Tooltip, 珊瑚暴力为固定300ms，不可调节！
  SetTimer, RemoveTooltip, -1500
  return
}
Gui, Delay:Submit, NoHide
BaoLiHoldTime := DBaoLiHoldEdit + 0
if (BaoLiHoldTime < 50)
  BaoLiHoldTime := 50
if (BaoLiHoldTime > 500)
  BaoLiHoldTime := 500
GuiControl, Main:, BindStatus, 暴力按住时长：%BaoLiHoldTime%ms
Tooltip, 暴力按住时长已更新：%BaoLiHoldTime%ms
SetTimer, RemoveTooltip, -1500
return
OnBaoLiVersionChangeFromWindow:
GuiControlGet, IsShanhu,, DBVShanhu
if (IsShanhu = 1) {
  BaoLiVersion := "珊瑚"
  GuiControl, Delay:Disable, DBaoLiHoldEdit
  UpdateBaoLiVersionUI()
  Tooltip, 已切换到：珊瑚暴力（固定）`n按住时长固定300ms
} else {
  BaoLiVersion := "酒窝"
  GuiControl, Delay:Enable, DBaoLiHoldEdit
  UpdateBaoLiVersionUI()
  Tooltip, 已切换到：酒窝暴力（可调）`n按住时长可调（50-500ms）
}
SetTimer, RemoveTooltip, -1500
return
```

## 十九、键位映射窗口

漂移/氮气/小喷/前进四个驾驶键位的编辑窗口：保存时逐项校验合法性，支持恢复默认与关闭销毁。

```ahk
OpenKeyMappingWindow:
if (KeyMapWindowOpen) {
  Gui, KeyMap:Show
  return
}
KeyMapWindowOpen := true
Gui, KeyMap:New, +AlwaysOnTop +ToolWindow +OwnerMain
Gui, KeyMap:Color, F0F5FA, FFFFFF
Gui, KeyMap:Font, s11 c222222, 微软雅黑
Gui, KeyMap:Margin, 15, 15
Gui, KeyMap:Add, Text, x0 y0 w340 h30 Center c0055BB, ★ 全局键位映射设置
Gui, KeyMap:Font, s10 c222222
Gui, KeyMap:Add, Text, x20 y45 w70 h22, 漂移键：
Gui, KeyMap:Add, Edit, x95 y42 w60 h24 vKMP_Drift, %DriftKey%
Gui, KeyMap:Add, Text, x170 y45 w70 h22, 氮气键：
Gui, KeyMap:Add, Edit, x245 y42 w60 h24 vKMP_Nitro, %NitroKey%
Gui, KeyMap:Add, Text, x20 y80 w70 h22, 小喷键：
Gui, KeyMap:Add, Edit, x95 y77 w60 h24 vKMP_Boost, %BoostKey%
Gui, KeyMap:Add, Text, x170 y80 w70 h22, 前进键：
Gui, KeyMap:Add, Edit, x245 y77 w60 h24 vKMP_Forward, %ForwardKey%
Gui, KeyMap:Add, Text, x20 y115 w300 h20 c666666 Center, 修改后点击"保存"立即生效
Gui, KeyMap:Add, Button, x60 y145 w70 h28 gApplyKeyMappingFromWindow, 保存
Gui, KeyMap:Add, Button, x140 y145 w80 h28 gResetKeyMapping, 恢复默认
Gui, KeyMap:Add, Button, x230 y145 w70 h28 gCloseKeyMapWindow, 取消
Gui, KeyMap:Show, xCenter yCenter w360 h200, 全局键位映射
return
ApplyKeyMappingFromWindow:
Gui, KeyMap:Submit, NoHide
if (!IsKeyValid(KMP_Drift)) {
  MsgBox, 48,, 漂移键 "%KMP_Drift%" 无效！`n请输入正确的按键名称。
  return
}
if (!IsKeyValid(KMP_Nitro)) {
  MsgBox, 48,, 氮气键 "%KMP_Nitro%" 无效！`n请输入正确的按键名称。
  return
}
if (!IsKeyValid(KMP_Boost)) {
  MsgBox, 48,, 小喷键 "%KMP_Boost%" 无效！`n请输入正确的按键名称。
  return
}
if (!IsKeyValid(KMP_Forward)) {
  MsgBox, 48,, 前进键 "%KMP_Forward%" 无效！`n请输入正确的按键名称。
  return
}
DriftKey := KMP_Drift
NitroKey := KMP_Nitro
BoostKey := KMP_Boost
ForwardKey := KMP_Forward
GuiControl, Main:, BindStatus, 键位已更新：漂移=%DriftKey% 氮气=%NitroKey% 小喷=%BoostKey% 前进=%ForwardKey%
Tooltip, 键位映射已更新！
SetTimer, RemoveTooltip, -2000
Gui, KeyMap:Destroy
KeyMapWindowOpen := false
return
ResetKeyMapping:
GuiControl, KeyMap:, KMP_Drift, q
GuiControl, KeyMap:, KMP_Nitro, e
GuiControl, KeyMap:, KMP_Boost, Up
GuiControl, KeyMap:, KMP_Forward, w
Tooltip, 已恢复默认键位
SetTimer, RemoveTooltip, -1500
return
CloseKeyMapWindow:
Gui, KeyMap:Destroy
KeyMapWindowOpen := false
return
KeyMapGuiClose:
Gui, KeyMap:Destroy
KeyMapWindowOpen := false
return
```

## 二十、延迟参数设置窗口

集中编辑切断、一段/二段连卡、三四五连卡的各段延时，以及连卡数量、模式、暴力版本与按住时长。

```ahk
OpenDelaySettingsWindow:
if (DelayWindowOpen) {
  Gui, Delay:Show
  return
}
DelayWindowOpen := true
Gui, Delay:New, +AlwaysOnTop +ToolWindow +OwnerMain
Gui, Delay:Color, F0F5FA, FFFFFF
Gui, Delay:Font, s11 c222222, 微软雅黑
Gui, Delay:Margin, 15, 15
Gui, Delay:Add, Text, x0 y0 w440 h30 Center c0055BB, ★ 延迟参数设置
Gui, Delay:Font, s10 c222222
Gui, Delay:Add, Text, x20 y45 w70 h20, 快速切断：
Gui, Delay:Add, Text, x95 y45 w40 h20, 按住
Gui, Delay:Add, Edit, x140 y42 w50 h24 vDCutHoldEdit, %CutHoldTime%
Gui, Delay:Add, UpDown, x140 y42 w50 h24 vDCutHoldUD Range5-100, %CutHoldTime%
Gui, Delay:Add, Text, x200 y45 w40 h20, 间隔
Gui, Delay:Add, Edit, x245 y42 w50 h24 vDCutIntervalEdit, %CutInterval%
Gui, Delay:Add, UpDown, x245 y42 w50 h24 vDCutIntervalUD Range5-100, %CutInterval%
Gui, Delay:Add, Button, x310 y40 w70 h24 gApplyCutDelayFromWindow, 应用
Gui, Delay:Add, Text, x20 y80 w70 h20, 一段连卡：
Gui, Delay:Add, Text, x95 y80 w40 h20, 按住
Gui, Delay:Add, Edit, x140 y77 w50 h24 vDK1HoldEdit, %KaQi1HoldTime%
Gui, Delay:Add, UpDown, x140 y77 w50 h24 vDK1HoldUD Range5-100, %KaQi1HoldTime%
Gui, Delay:Add, Text, x200 y80 w40 h20, 保底
Gui, Delay:Add, Edit, x245 y77 w50 h24 vDK1MinEdit, %KaQi1MinTotal%
Gui, Delay:Add, UpDown, x245 y77 w50 h24 vDK1MinUD Range5-100, %KaQi1MinTotal%
Gui, Delay:Add, Button, x310 y75 w70 h24 gApplyK1DelayFromWindow, 应用
Gui, Delay:Add, Text, x20 y115 w70 h20, 二段连卡：
Gui, Delay:Add, Text, x95 y115 w40 h20, 一段
Gui, Delay:Add, Edit, x140 y112 w50 h24 vDK2FirstEdit, %KaQi2FirstHold%
Gui, Delay:Add, UpDown, x140 y112 w50 h24 vDK2FirstUD Range5-100, %KaQi2FirstHold%
Gui, Delay:Add, Text, x200 y115 w40 h20, 等待
Gui, Delay:Add, Edit, x245 y112 w50 h24 vDK2WaitEdit, %KaQi2WaitTime%
Gui, Delay:Add, UpDown, x245 y112 w50 h24 vDK2WaitUD Range100-2000, %KaQi2WaitTime%
Gui, Delay:Add, Text, x305 y115 w40 h20, 二段
Gui, Delay:Add, Edit, x350 y112 w50 h24 vDK2SecondEdit, %KaQi2SecondHold%
Gui, Delay:Add, UpDown, x350 y112 w50 h24 vDK2SecondUD Range5-100, %KaQi2SecondHold%
Gui, Delay:Add, Button, x415 y110 w70 h24 gApplyK2DelayFromWindow, 应用
Gui, Delay:Add, Text, x20 y150 w95 h20, 三四五连卡：
Gui, Delay:Add, Text, x95 y150 w40 h20, 首按
Gui, Delay:Add, Edit, x140 y147 w50 h24 vDLKFirstEdit, %LianKaFirstHold%
Gui, Delay:Add, UpDown, x140 y147 w50 h24 vDLKFirstUD Range5-100, %LianKaFirstHold%
Gui, Delay:Add, Text, x200 y150 w40 h20, 间隔
Gui, Delay:Add, Edit, x245 y147 w50 h24 vDLKWaitEdit, %LianKaWaitTime%
Gui, Delay:Add, UpDown, x245 y147 w50 h24 vDLKWaitUD Range100-2000, %LianKaWaitTime%
Gui, Delay:Add, Text, x305 y150 w40 h20, 连按
Gui, Delay:Add, Edit, x350 y147 w50 h24 vDLKSecondEdit, %LianKaSecondHold%
Gui, Delay:Add, UpDown, x350 y147 w50 h24 vDLKSecondUD Range5-100, %LianKaSecondHold%
Gui, Delay:Add, Button, x415 y147 w70 h24 gApplyLianKaDelayFromWindow, 应用
Gui, Delay:Add, Text, x20 y178 w50 h20, 连数：
Gui, Delay:Add, Radio, x70 y175 w60 h20 Group vDLK3 gOnLianKaCountChange, 三连
Gui, Delay:Add, Radio, x135 y175 w60 h20 vDLK4 gOnLianKaCountChange, 四连
Gui, Delay:Add, Radio, x200 y175 w60 h20 vDLK5 gOnLianKaCountChange, 五连
Gui, Delay:Add, CheckBox, x280 y175 w210 h20 vDLianKaModeCB gOnLianKaModeChange, 首段带小喷（模式B）
Gui, Delay:Add, Text, x20 y210 w70 h20, 暴力版本：
Gui, Delay:Add, Radio, x95 y207 w150 h20 Group -Wrap vDBVShanhu gOnBaoLiVersionChangeFromWindow, 珊瑚（固定300ms）
Gui, Delay:Add, Radio, x245 y207 w200 h20 -Wrap vDBVJiuwo gOnBaoLiVersionChangeFromWindow, 酒窝（可调50-500ms）
Gui, Delay:Add, Text, x20 y238 w80 h20, 按住时长：
Gui, Delay:Add, Edit, x105 y235 w50 h24 vDBaoLiHoldEdit, %BaoLiHoldTime%
Gui, Delay:Add, UpDown, x105 y235 w50 h24 vDBaoLiHoldUD Range50-500, %BaoLiHoldTime%
Gui, Delay:Add, Text, x160 y238 w25 h20, ms
Gui, Delay:Add, Button, x310 y235 w70 h24 gApplyBaoLiHoldFromWindow, 应用
Gui, Delay:Add, Button, x170 y275 w100 h28 gCloseDelayWindow, 关闭
if (BaoLiVersion = "珊瑚") {
  GuiControl, Delay:, DBVShanhu, 1
  GuiControl, Delay:Disable, DBaoLiHoldEdit
} else {
  GuiControl, Delay:, DBVJiuwo, 1
  GuiControl, Delay:Enable, DBaoLiHoldEdit
}
if (LianKaCount = 3) {
  GuiControl, Delay:, DLK3, 1
} else if (LianKaCount = 4) {
  GuiControl, Delay:, DLK4, 1
} else {
  GuiControl, Delay:, DLK5, 1
}
if (LianKaMode = 2)
  GuiControl, Delay:, DLianKaModeCB, 1
else
  GuiControl, Delay:, DLianKaModeCB, 0
Gui, Delay:Show, xCenter yCenter w500 h315, 延迟参数设置
return
CloseDelayWindow:
Gui, Delay:Destroy
DelayWindowOpen := false
return
DelayGuiClose:
Gui, Delay:Destroy
DelayWindowOpen := false
return
```

## 二十一、配置总览窗口

只读展示当前键位映射、延迟参数、连卡数量与模式、暴力版本及按住时长，支持手动刷新。

```ahk
OpenOverviewWindow:
if (OverviewWindowOpen) {
  Gui, Overview:Show
  return
}
OverviewWindowOpen := true
Gui, Overview:New, +AlwaysOnTop +ToolWindow +OwnerMain
Gui, Overview:Color, F0F5FA, FFFFFF
Gui, Overview:Font, s11 c222222, 微软雅黑
Gui, Overview:Margin, 15, 15
Gui, Overview:Add, Text, x0 y0 w440 h35 Center c0055BB, ★ 当前配置总览
Gui, Overview:Font, s10 c222222
Gui, Overview:Add, Text, x15 y45 w150 h22, 【键位映射】
Gui, Overview:Add, Text, x15 y70 w70 h22, 漂移键：
Gui, Overview:Add, Text, x90 y70 w60 h22 vOV_Drift, %DriftKey%
Gui, Overview:Add, Text, x170 y70 w70 h22, 氮气键：
Gui, Overview:Add, Text, x245 y70 w60 h22 vOV_Nitro, %NitroKey%
Gui, Overview:Add, Text, x15 y100 w70 h22, 小喷键：
Gui, Overview:Add, Text, x90 y100 w60 h22 vOV_Boost, %BoostKey%
Gui, Overview:Add, Text, x170 y100 w70 h22, 前进键：
Gui, Overview:Add, Text, x245 y100 w60 h22 vOV_Forward, %ForwardKey%
Gui, Overview:Add, Text, x15 y135 w150 h22, 【延迟参数】
Gui, Overview:Add, Text, x15 y160 w80 h20, 快速切断：
Gui, Overview:Add, Text, x100 y160 w150 h20 vOV_Cut, 按住 %CutHoldTime%ms / 间隔 %CutInterval%ms
Gui, Overview:Add, Text, x15 y185 w80 h20, 一段连卡：
Gui, Overview:Add, Text, x100 y185 w150 h20 vOV_K1, 按住 %KaQi1HoldTime%ms / 保底 %KaQi1MinTotal%ms
Gui, Overview:Add, Text, x15 y210 w80 h20, 二段连卡：
Gui, Overview:Add, Text, x100 y210 w230 h20 vOV_K2, 一段 %KaQi2FirstHold%ms / 等待 %KaQi2WaitTime%ms / 二段 %KaQi2SecondHold%ms
LianKaCountDesc := (LianKaCount = 3 ? "三连" : LianKaCount = 4 ? "四连" : "五连")
LianKaModeDesc := (LianKaMode = 2 ? "模式B（首段带小喷）" : "模式A（先漂移）")
Gui, Overview:Add, Text, x15 y235 w95 h20, 三四五连卡：
Gui, Overview:Add, Text, x100 y235 w280 h20 vOV_LianKa, 首按 %LianKaFirstHold%ms / 间隔 %LianKaWaitTime%ms / 连按 %LianKaSecondHold%ms
Gui, Overview:Add, Text, x15 y258 w220 h20 cE60000 vOV_LianKaCount, 当前连卡：%LianKaCountDesc%（%LianKaModeDesc%）
Gui, Overview:Add, Text, x15 y285 w150 h22, 【暴力模式】
Gui, Overview:Add, Text, x15 y310 w80 h20, 当前版本：
OV_VersionText := (BaoLiVersion = "珊瑚" ? "珊瑚暴力（固定300ms）" : "酒窝暴力（可调50-500ms）")
Gui, Overview:Add, Text, x100 y310 w180 h20 vOV_Version, %OV_VersionText%
Gui, Overview:Add, Text, x15 y335 w80 h20, 按住时长：
Gui, Overview:Add, Text, x100 y335 w80 h20 vOV_HoldTime, %BaoLiHoldTime%ms
Gui, Overview:Add, Button, x120 y380 w90 h28 gRefreshOverview, 刷新
Gui, Overview:Add, Button, x220 y380 w90 h28 gCloseOverviewWindow, 关闭
Gui, Overview:Show, xCenter yCenter w460 h435, 当前配置总览
return
RefreshOverview:
GuiControl, Overview:, OV_Drift, %DriftKey%
GuiControl, Overview:, OV_Nitro, %NitroKey%
GuiControl, Overview:, OV_Boost, %BoostKey%
GuiControl, Overview:, OV_Forward, %ForwardKey%
GuiControl, Overview:, OV_Cut, 按住 %CutHoldTime%ms / 间隔 %CutInterval%ms
GuiControl, Overview:, OV_K1, 按住 %KaQi1HoldTime%ms / 保底 %KaQi1MinTotal%ms
GuiControl, Overview:, OV_K2, 一段 %KaQi2FirstHold%ms / 等待 %KaQi2WaitTime%ms / 二段 %KaQi2SecondHold%ms
LianKaCountDesc := (LianKaCount = 3 ? "三连" : LianKaCount = 4 ? "四连" : "五连")
LianKaModeDesc := (LianKaMode = 2 ? "模式B（首段带小喷）" : "模式A（先漂移）")
GuiControl, Overview:, OV_LianKa, 首按 %LianKaFirstHold%ms / 间隔 %LianKaWaitTime%ms / 连按 %LianKaSecondHold%ms
GuiControl, Overview:, OV_LianKaCount, 当前连卡：%LianKaCountDesc%（%LianKaModeDesc%）
OV_VersionText := (BaoLiVersion = "珊瑚" ? "珊瑚暴力（固定300ms）" : "酒窝暴力（可调50-500ms）")
GuiControl, Overview:, OV_Version, %OV_VersionText%
GuiControl, Overview:, OV_HoldTime, %BaoLiHoldTime%ms
Tooltip, 已刷新
SetTimer, RemoveTooltip, -1000
return
CloseOverviewWindow:
Gui, Overview:Destroy
OverviewWindowOpen := false
return
OverviewGuiClose:
Gui, Overview:Destroy
OverviewWindowOpen := false
return
```

## 二十二、F 功能键窗口

F10 快速切断、F11 三四五连卡、F12 暴力出弯的单独绑定与启停界面。

```ahk
OpenFKeyWindow:
if (FKeyWindowOpen) {
  Gui, FKey:Show
  return
}
FKeyWindowOpen := true
Gui, FKey:New, +AlwaysOnTop +ToolWindow +OwnerMain
Gui, FKey:Color, F0F5FA, FFFFFF
Gui, FKey:Font, s11 c222222, 微软雅黑
Gui, FKey:Margin, 15, 15
Gui, FKey:Add, Text, x0 y0 w450 h30 Center c0055BB, ★ F功能键控制
Gui, FKey:Font, s10 c222222
Gui, FKey:Add, Text, x20 y45 w120 h22, F10 快速切断：
Gui, FKey:Add, Edit, x150 y42 w60 h24 vF10Edit, %Hotkey_F10%
Gui, FKey:Add, Button, x220 y42 w60 h24 gBindF10, 绑定
Gui, FKey:Add, Button, x285 y42 w60 h24 gToggleF10, 启停
Gui, FKey:Add, Text, x355 y45 w80 h22 vStateF10 cFF0000, 未启用
Gui, FKey:Add, Text, x20 y80 w120 h22, F11 三四五连卡：
Gui, FKey:Add, Edit, x150 y77 w60 h24 vF11Edit, %Hotkey_F11%
Gui, FKey:Add, Button, x220 y77 w60 h24 gBindF11, 绑定
Gui, FKey:Add, Button, x285 y77 w60 h24 gToggleF11, 启停
Gui, FKey:Add, Text, x355 y80 w80 h22 vStateF11 cFF0000, 未启用
Gui, FKey:Add, Text, x20 y115 w120 h22, F12 暴力出弯：
Gui, FKey:Add, Edit, x150 y112 w60 h24 vF12Edit, %Hotkey_F12%
Gui, FKey:Add, Button, x220 y112 w60 h24 gBindF12, 绑定
Gui, FKey:Add, Button, x285 y112 w60 h24 gToggleF12, 启停
Gui, FKey:Add, Text, x355 y115 w80 h22 vStateF12 cFF0000, 未启用
Gui, FKey:Add, Text, x20 y150 w410 h20 c666666, 按 F10/F11/F12 可单独启停对应功能
Gui, FKey:Add, Button, x175 y180 w100 h28 gCloseFKeyWindow, 关闭
UpdateFKeyStates()
Gui, FKey:Show, xCenter yCenter w450 h220, F功能键控制
return
CloseFKeyWindow:
Gui, FKey:Destroy
FKeyWindowOpen := false
return
FKeyGuiClose:
Gui, FKey:Destroy
FKeyWindowOpen := false
return
```

## 二十三、游戏窗口检测与热键分发 FuncWrapper

IsGameActive 判断前台是否为游戏进程；FuncWrapper 为所有热键统一入口，未在游戏内时原样透传按键，游戏内按启用状态分发到对应宏动作。

:::encrypted{password='Game'}
IsGameActive() {
  return WinActive("ahk_exe GameApp.exe")
}
FuncWrapper:
ThisHotkey := A_ThisHotkey
if !IsGameActive() {
  RawKey := RegExReplace(ThisHotkey, "^[\$\*~&<>!^+#]", "")
  if (RawKey = "Left" || RawKey = "Down" || RawKey = "Right") {
    Send, {Blind}{%RawKey%}
  } else {
    Send, {Blind}{%RawKey%}
  }
  return
}
RawKey := RegExReplace(ThisHotkey, "^[\$\*~&<>!^+#]", "")
if (RawKey = Hotkey_BaoLi && baoLiOn) {
  Gosub, BaoLiAction
}
else if (RawKey = Hotkey_WuHen && wuHenOn) {
  Gosub, WuHenAction
}
else if (RawKey = Hotkey_Cut && spaceCut) {
  Gosub, CutAction
}
else if (RawKey = Hotkey_KaQi1 && kaQi1On) {
  Gosub, KaQi1Action
}
else if (RawKey = Hotkey_KaQi2 && kaQi2On) {
  Gosub, KaQi2Action
}
else if (RawKey = Hotkey_LianKa && lianKaOn) {
  Gosub, LianKaAction
}
else if (RawKey = "Left") {
  Gosub, LeftAction
}
else if (RawKey = "Down") {
  Gosub, DownAction
}
else if (RawKey = "Right") {
  Gosub, RightAction
}
else if (RawKey = "3" || RawKey = "4" || RawKey = "5") {
  LianKaSwitchKey := RawKey
  Gosub, LianKaSwitchHotkey
}
else {
  Send, {Blind}{%RawKey%}
}
return
:::

## 二十四、总开关 F1 / 隐藏 F2 / 重置 F3

F1 全部开/全部关；F2 切换主窗口显示隐藏；F3 强制把所有功能重新开启。

```ahk
F1Action:
allOn := (baoLiOn && wuHenOn && spaceCut && kaQi1On && kaQi2On && lianKaOn)
if (allOn) {
  if (baoLiOn) {
    baoLiOn := false
    Hotkey, $%Hotkey_BaoLi%, FuncWrapper, Off
    SetStateColor("StateB", "未启用", "FF0000")
  }
  if (wuHenOn) {
    wuHenOn := false
    Hotkey, $%Hotkey_WuHen%, FuncWrapper, Off
    SetStateColor("StateW", "未启用", "FF0000")
  }
  if (spaceCut) {
    spaceCut := false
    Hotkey, %Hotkey_Cut%, FuncWrapper, Off
    SetStateColor("StateS", "未启用", "FF0000")
  }
  if (kaQi1On) {
    kaQi1On := false
    if (Hotkey_KaQi1 != "Left" && Hotkey_KaQi1 != "Down") {
      Hotkey, $%Hotkey_KaQi1%, FuncWrapper, Off
    }
    SetStateColor("StateK1", "未启用", "FF0000")
  }
  if (kaQi2On) {
    kaQi2On := false
    if (Hotkey_KaQi2 != "Left" && Hotkey_KaQi2 != "Down") {
      Hotkey, $%Hotkey_KaQi2%, FuncWrapper, Off
    }
    SetStateColor("StateK2", "未启用", "FF0000")
  }
  if (lianKaOn) {
    lianKaOn := false
    if (Hotkey_LianKa != "Right") {
      Hotkey, $%Hotkey_LianKa%, FuncWrapper, Off
    }
    SetStateColor("StateLK", "未启用", "FF0000")
  }
  Send, {%NitroKey% up}
  Send, {%ForwardKey% up}
  Send, {%BoostKey% up}
  Send, {Ctrl up}
  Send, {%DriftKey% up}
  Tooltip, 所有功能已全部关闭
  GuiControl, Main:, BindStatus, [已关闭] 全部已关闭
} else {
  if (!baoLiOn) {
    baoLiOn := true
    Hotkey, $%Hotkey_BaoLi%, FuncWrapper, On
    SetStateColor("StateB", "已启用", "009933")
  }
  if (!wuHenOn) {
    wuHenOn := true
    Hotkey, $%Hotkey_WuHen%, FuncWrapper, On
    SetStateColor("StateW", "已启用", "009933")
  }
  if (!spaceCut) {
    spaceCut := true
    Hotkey, %Hotkey_Cut%, FuncWrapper, On
    SetStateColor("StateS", "已启用", "009933")
  }
  if (!kaQi1On) {
    kaQi1On := true
    if (Hotkey_KaQi1 != "Left" && Hotkey_KaQi1 != "Down") {
      Hotkey, $%Hotkey_KaQi1%, FuncWrapper, On
    }
    SetStateColor("StateK1", "已启用", "009933")
  }
  if (!kaQi2On) {
    kaQi2On := true
    if (Hotkey_KaQi2 != "Left" && Hotkey_KaQi2 != "Down") {
      Hotkey, $%Hotkey_KaQi2%, FuncWrapper, On
    }
    SetStateColor("StateK2", "已启用", "009933")
  }
  if (!lianKaOn) {
    lianKaOn := true
    if (Hotkey_LianKa != "Right") {
      Hotkey, $%Hotkey_LianKa%, FuncWrapper, On
    }
    SetStateColor("StateLK", "已启用", "009933")
  }
  Tooltip, 所有功能已全部开启
  GuiControl, Main:, BindStatus, [已开启] 全部已开启
}
UpdateFKeyStates()
SetTimer, RemoveTooltip, -1500
return
F2Action:
guiShow := !guiShow
if (guiShow)
  Gui, Main:Show, xCenter yCenter
else
  Gui, Main:Hide
return
F3Action:
if (!baoLiOn) {
  baoLiOn := true
  Hotkey, $%Hotkey_BaoLi%, FuncWrapper, On
  SetStateColor("StateB", "已启用", "009933")
}
if (!wuHenOn) {
  wuHenOn := true
  Hotkey, $%Hotkey_WuHen%, FuncWrapper, On
  SetStateColor("StateW", "已启用", "009933")
}
if (!spaceCut) {
  spaceCut := true
  Hotkey, %Hotkey_Cut%, FuncWrapper, On
  SetStateColor("StateS", "已启用", "009933")
}
if (!kaQi1On) {
  kaQi1On := true
  if (Hotkey_KaQi1 != "Left" && Hotkey_KaQi1 != "Down") {
    Hotkey, $%Hotkey_KaQi1%, FuncWrapper, On
  }
  SetStateColor("StateK1", "已启用", "009933")
}
if (!kaQi2On) {
  kaQi2On := true
  if (Hotkey_KaQi2 != "Left" && Hotkey_KaQi2 != "Down") {
    Hotkey, $%Hotkey_KaQi2%, FuncWrapper, On
  }
  SetStateColor("StateK2", "已启用", "009933")
}
if (!lianKaOn) {
  lianKaOn := true
  if (Hotkey_LianKa != "Right") {
    Hotkey, $%Hotkey_LianKa%, FuncWrapper, On
  }
  SetStateColor("StateLK", "已启用", "009933")
}
Tooltip, 所有功能已全部开启（已重置）
GuiControl, Main:, BindStatus, [已开启] 全部已开启（重置）
UpdateFKeyStates()
SetTimer, RemoveTooltip, -1500
return
```

## 二十五、F10 / F11 / F12 动作与窗口检测

F10/F11/F12 分别调用对应启停函数；CheckWindow 定时器每 500ms 更新主窗口的游戏激活状态提示。

```ahk
F10Action:
ToggleCutFunc()
return
F11Action:
ToggleLianKaFunc(true)
return
F12Action:
ToggleBaoLiFunc()
return
CheckWindow:
if WinActive("ahk_exe GameApp.exe") {
  GuiControl, Main:, WindowStatus, 游戏窗口已激活 (GameApp.exe)
  GuiControl, Main:+cGreen, WindowStatus
} else {
  GuiControl, Main:, WindowStatus, 请切换到游戏窗口 (GameApp.exe)
  GuiControl, Main:+cRed, WindowStatus
}
return
```

## 二十六、通用宏解析 RunIniMacro()

解析以 @ 分段、| 分隔动作的宏数据，支持 send 按键（替换四个驾驶键位占位符）与 sleep 延时两类动作。

```ahk
RunIniMacro(MacroData) {
  if !IsGameActive()
    return
  StringSplit, Parts, MacroData, @
  if (Parts0 < 2) {
    return
  }
  ActionList := Parts2
  Actions := []
  Loop, Parse, ActionList, |
  {
    Actions.Push(A_LoopField)
  }
  for index, Action in Actions
  {
    Action := Trim(Action)
    if (Action = "")
      continue
    if (SubStr(Action, 1, 4) = "send") {
      KeyPart := SubStr(Action, 6)
      KeyPart := StrReplace(KeyPart, "{DriftKey}", DriftKey)
      KeyPart := StrReplace(KeyPart, "{NitroKey}", NitroKey)
      KeyPart := StrReplace(KeyPart, "{BoostKey}", BoostKey)
      KeyPart := StrReplace(KeyPart, "{ForwardKey}", ForwardKey)
      Send, %KeyPart%
    }
    else if (SubStr(Action, 1, 5) = "sleep") {
      Ms := SubStr(Action, 7)
      Ms := Ms + 0
      if (Ms > 0)
        Sleep, %Ms%
    }
    else {
    }
  }
}
```

## 二十七、核心宏动作序列

暴力出弯（珊瑚固定 300ms / 酒窝可调）、无痕三喷、快速切断、一段/二段卡气、三四五连卡（模式 A/B）等具体按键时序。

:::encrypted{password='Game'}
BaoLiAction:
if (BaoLiRunning)
  return
BaoLiRunning := true
if (BaoLiVersion = "珊瑚") {
  Gosub, CoralBaoLi
} else {
  Gosub, JiuWoBaoLi
}
BaoLiRunning := false
return
CoralBaoLi:
Send, {Blind}{%NitroKey% down}
Send, {Blind}{%BoostKey% down}
Sleep, 10
Send, {Blind}{%ForwardKey% down}
Sleep, 300
Send, {Blind}{%BoostKey% up}
Send, {Blind}{%NitroKey% up}
Sleep, 40
Send, {Blind}{%BoostKey%}
Sleep, 20
Send, {Blind}{%BoostKey%}
Sleep, 20
Send, {Blind}{%BoostKey%}
Sleep, 20
Send, {Blind}{%BoostKey%}
Sleep, 20
Send, {Blind}{%BoostKey%}
Sleep, 20
Send, {Blind}{%BoostKey%}
return
JiuWoBaoLi:
Random, s, 0, 50
Sleep, %s%
Send, {Blind}{%NitroKey% down}
Send, {Blind}{%BoostKey% down}
Sleep, 10
Send, {Blind}{%ForwardKey% down}
Sleep, %BaoLiHoldTime%
Send, {Blind}{%BoostKey% up}
Send, {Blind}{%NitroKey% up}
return
WuHenAction:
Random, r1, 4, 12
Sleep, %r1%
SendInput, {Blind}{Ctrl down}
Sleep, 1
SendInput, {Blind}{%BoostKey% down}
Random, r2, 8, 20
Sleep, %r2%
SendInput, {Blind}{Ctrl up}
Sleep, 1
Random, r3, 5, 15
Sleep, %r3%
SendInput, {Blind}{%BoostKey% up}
return
CutAction:
Loop, 3 {
  SendInput, {%DriftKey% Down}
  PreciseSleep(CutHoldTime)
  SendInput, {%DriftKey% Up}
  if (A_Index < 3) {
    PreciseSleep(CutInterval)
  }
}
return
KaQi1Action:
SendInput, {%BoostKey% Down}{%DriftKey% Down}
PreciseSleep(KaQi1HoldTime)
SendInput, {%BoostKey% Up}{%DriftKey% Up}
DllCall("QueryPerformanceCounter", "Int64*", startCounter)
DllCall("QueryPerformanceCounter", "Int64*", endCounter)
elapsed := ((endCounter - startCounter) * 1000) / Frequency
if (elapsed < KaQi1MinTotal)
  PreciseSleep(KaQi1MinTotal - elapsed)
return
KaQi2Action:
SendInput, {%DriftKey% Down}
PreciseSleep(KaQi2FirstHold)
SendInput, {%DriftKey% Up}
PreciseSleep(KaQi2WaitTime)
SendInput, {%BoostKey% Down}{%DriftKey% Down}
PreciseSleep(KaQi2SecondHold)
SendInput, {%BoostKey% Up}{%DriftKey% Up}
return
LianKaSwitchHotkey:
if (LianKaSwitchKey = "3")
  LianKaCount := 3
else if (LianKaSwitchKey = "4")
  LianKaCount := 4
else
  LianKaCount := 5
LianKaCountDesc := (LianKaCount = 3 ? "三连" : LianKaCount = 4 ? "四连" : "五连")
if (DelayWindowOpen) {
  if (LianKaCount = 3)
    GuiControl, Delay:, DLK3, 1
  else if (LianKaCount = 4)
    GuiControl, Delay:, DLK4, 1
  else
    GuiControl, Delay:, DLK5, 1
}
if (OverviewWindowOpen)
  RefreshOverview()
GuiControl, Main:, BindStatus, 三四五连卡：首按 %LianKaFirstHold%ms / 间隔 %LianKaWaitTime%ms / 连按 %LianKaSecondHold%ms / %LianKaCountDesc%
Tooltip, 连卡数量已切换为：%LianKaCountDesc%
SetTimer, RemoveTooltip, -1500
return
LianKaAction:
if (LianKaRunning)
  return
LianKaRunning := true
if (LianKaMode = 2) {
  Loop, %LianKaCount% {
    SendInput, {%BoostKey% Down}{%DriftKey% Down}
    if (A_Index = 1)
      PreciseSleep(LianKaFirstHold)
    else
      PreciseSleep(LianKaSecondHold)
    SendInput, {%BoostKey% Up}{%DriftKey% Up}
    if (A_Index < LianKaCount)
      PreciseSleep(LianKaWaitTime)
  }
} else {
  SendInput, {%DriftKey% Down}
  PreciseSleep(LianKaFirstHold)
  SendInput, {%DriftKey% Up}
  Loop, % LianKaCount - 1 {
    PreciseSleep(LianKaWaitTime)
    SendInput, {%BoostKey% Down}{%DriftKey% Down}
    PreciseSleep(LianKaSecondHold)
    SendInput, {%BoostKey% Up}{%DriftKey% Up}
  }
}
LianKaRunning := false
return
:::

## 二十八、方向键动作 Left / Down / Right

方向键优先转发到已启用的对应卡气/连卡功能，否则原样发送方向键。

```ahk
LeftAction:
if (kaQi1On) {
  if (Hotkey_KaQi1 = "Left") {
    Gosub, KaQi1Action
    return
  }
}
Send, {Blind}{Left}
return
DownAction:
if (kaQi2On) {
  if (Hotkey_KaQi2 = "Down") {
    Gosub, KaQi2Action
    return
  }
}
Send, {Blind}{Down}
return
RightAction:
if (lianKaOn) {
  if (Hotkey_LianKa = "Right") {
    Gosub, LianKaAction
    return
  }
}
Send, {Blind}{Right}
return
```

## 二十九、六大功能键绑定回调

读取输入框新键位，校验合法性后解绑旧键、绑定新键；提示按 F3 使新键位生效。

```ahk
BindBaoLi:
GuiControlGet, n,, BaoLiEdit
if (n = Hotkey_BaoLi) {
  MsgBox, 262144,, 该键位已经是暴力出弯的键位，无需重复绑定！
  return
}
if (!IsKeyValid(n)) {
  MsgBox, 48,, 按键 "%n%" 无效！`n请输入正确的按键名称。
  return
}
try Hotkey, $%Hotkey_BaoLi%, FuncWrapper, Off
  Hotkey_BaoLi := n
if (baoLiOn)
  Hotkey, $%Hotkey_BaoLi%, FuncWrapper, On
GuiControl, Main:, BindStatus, 暴力出弯已绑定为：%n%
MsgBox, 262144,, 暴力出弯已绑定为：%n%`n按 F3 使新键位生效
return
BindWuHen:
GuiControlGet, n,, WuHenEdit
if (n = Hotkey_WuHen) {
  MsgBox, 262144,, 该键位已经是无痕三喷的键位，无需重复绑定！
  return
}
if (!IsKeyValid(n)) {
  MsgBox, 48,, 按键 "%n%" 无效！`n请输入正确的按键名称。
  return
}
try Hotkey, $%Hotkey_WuHen%, FuncWrapper, Off
  Hotkey_WuHen := n
if (wuHenOn)
  Hotkey, $%Hotkey_WuHen%, FuncWrapper, On
GuiControl, Main:, BindStatus, 无痕三喷已绑定为：%n%
MsgBox, 262144,, 无痕三喷已绑定为：%n%`n按 F3 使新键位生效
return
BindSpace:
GuiControlGet, n,, SpaceEdit
if (n = Hotkey_Cut) {
  MsgBox, 262144,, 该键位已经是快速切断的键位，无需重复绑定！
  return
}
if (!IsKeyValid(n)) {
  MsgBox, 48,, 按键 "%n%" 无效！`n请输入正确的按键名称。
  return
}
try Hotkey, %Hotkey_Cut%, FuncWrapper, Off
  Hotkey_Cut := n
if (spaceCut)
  Hotkey, %Hotkey_Cut%, FuncWrapper, On
GuiControl, Main:, BindStatus, 快速切断已绑定为：%n%
MsgBox, 262144,, 快速切断已绑定为：%n%`n按 F3 使新键位生效
return
BindKaQi1:
GuiControlGet, n,, KaQi1Edit
if (n = Hotkey_KaQi1) {
  MsgBox, 262144,, 该键位已经是一段连卡的键位，无需重复绑定！
  return
}
if (!IsKeyValid(n)) {
  MsgBox, 48,, 按键 "%n%" 无效！`n请输入正确的按键名称。
  return
}
if (Hotkey_KaQi1 != "Left" && Hotkey_KaQi1 != "Down") {
  try Hotkey, $%Hotkey_KaQi1%, FuncWrapper, Off
  }
Hotkey_KaQi1 := n
if (n != "Left" && n != "Down") {
  if (kaQi1On)
    Hotkey, $%n%, FuncWrapper, On
  else
    Hotkey, $%n%, FuncWrapper, Off
}
GuiControl, Main:, BindStatus, 一段连卡已绑定为：%n%
MsgBox, 262144,, 一段连卡已绑定为：%n%`n按 F3 使新键位生效
return
BindKaQi2:
GuiControlGet, n,, KaQi2Edit
if (n = Hotkey_KaQi2) {
  MsgBox, 262144,, 该键位已经是二段连卡的键位，无需重复绑定！
  return
}
if (!IsKeyValid(n)) {
  MsgBox, 48,, 按键 "%n%" 无效！`n请输入正确的按键名称。
  return
}
if (Hotkey_KaQi2 != "Left" && Hotkey_KaQi2 != "Down") {
  try Hotkey, $%Hotkey_KaQi2%, FuncWrapper, Off
  }
Hotkey_KaQi2 := n
if (n != "Left" && n != "Down") {
  if (kaQi2On)
    Hotkey, $%n%, FuncWrapper, On
  else
    Hotkey, $%n%, FuncWrapper, Off
}
GuiControl, Main:, BindStatus, 二段连卡已绑定为：%n%
MsgBox, 262144,, 二段连卡已绑定为：%n%`n按 F3 使新键位生效
return
BindLianKa:
GuiControlGet, n,, LianKaEdit
if (n = Hotkey_LianKa) {
  MsgBox, 262144,, 该键位已经是三四五连卡的键位，无需重复绑定！
  return
}
if (!IsKeyValid(n)) {
  MsgBox, 48,, 按键 "%n%" 无效！`n请输入正确的按键名称。
  return
}
if (Hotkey_LianKa != "Right") {
  try Hotkey, $%Hotkey_LianKa%, FuncWrapper, Off
  }
Hotkey_LianKa := n
if (n != "Right") {
  if (lianKaOn)
    Hotkey, $%n%, FuncWrapper, On
  else
    Hotkey, $%n%, FuncWrapper, Off
}
GuiControl, Main:, BindStatus, 三四五连卡已绑定为：%n%
MsgBox, 262144,, 三四五连卡已绑定为：%n%`n按 F3 使新键位生效
return
```

## 三十、控制键绑定回调（F1/F2/F3/F10/F11/F12）

逐个校验控制键，禁止与其他控制键重复占用，解绑旧键并绑定新键，即时生效。

```ahk
BindF1:
GuiControlGet, n,, F1Edit
if (n = Hotkey_F1) {
  MsgBox, 262144,, 该键位已经是 F1 的键位，无需重复绑定！
  return
}
if (!IsKeyValid(n)) {
  MsgBox, 48,, 按键 "%n%" 无效！`n请输入正确的按键名称。
  return
}
if (n = Hotkey_F2 || n = Hotkey_F3 || n = Hotkey_F10 || n = Hotkey_F11 || n = Hotkey_F12) {
  MsgBox, 48,, 该键位已被其他控制键占用！
  return
}
try Hotkey, $%Hotkey_F1%, F1Action, Off
  Hotkey_F1 := n
Hotkey, $%Hotkey_F1%, F1Action, On
GuiControl, Main:, BindStatus, F1 已绑定为：%n%
MsgBox, 262144,, F1（总开关）已绑定为：%n%
return
BindF2:
GuiControlGet, n,, F2Edit
if (n = Hotkey_F2) {
  MsgBox, 262144,, 该键位已经是 F2 的键位，无需重复绑定！
  return
}
if (!IsKeyValid(n)) {
  MsgBox, 48,, 按键 "%n%" 无效！`n请输入正确的按键名称。
  return
}
if (n = Hotkey_F1 || n = Hotkey_F3 || n = Hotkey_F10 || n = Hotkey_F11 || n = Hotkey_F12) {
  MsgBox, 48,, 该键位已被其他控制键占用！
  return
}
try Hotkey, $%Hotkey_F2%, F2Action, Off
  Hotkey_F2 := n
Hotkey, $%Hotkey_F2%, F2Action, On
GuiControl, Main:, BindStatus, F2 已绑定为：%n%
MsgBox, 262144,, F2（隐藏窗口）已绑定为：%n%
return
BindF3:
GuiControlGet, n,, F3Edit
if (n = Hotkey_F3) {
  MsgBox, 262144,, 该键位已经是 F3 的键位，无需重复绑定！
  return
}
if (!IsKeyValid(n)) {
  MsgBox, 48,, 按键 "%n%" 无效！`n请输入正确的按键名称。
  return
}
if (n = Hotkey_F1 || n = Hotkey_F2 || n = Hotkey_F10 || n = Hotkey_F11 || n = Hotkey_F12) {
  MsgBox, 48,, 该键位已被其他控制键占用！
  return
}
try Hotkey, $%Hotkey_F3%, F3Action, Off
  Hotkey_F3 := n
Hotkey, $%Hotkey_F3%, F3Action, On
GuiControl, Main:, BindStatus, F3 已绑定为：%n%
MsgBox, 262144,, F3（强制重置）已绑定为：%n%
return
BindF10:
GuiControlGet, n,, F10Edit
if (n = Hotkey_F10) {
  MsgBox, 262144,, 该键位已经是 F10 的键位，无需重复绑定！
  return
}
if (!IsKeyValid(n)) {
  MsgBox, 48,, 按键 "%n%" 无效！`n请输入正确的按键名称。
  return
}
if (n = Hotkey_F1 || n = Hotkey_F2 || n = Hotkey_F3 || n = Hotkey_F11 || n = Hotkey_F12) {
  MsgBox, 48,, 该键位已被其他控制键占用！
  return
}
try Hotkey, $%Hotkey_F10%, F10Action, Off
  Hotkey_F10 := n
Hotkey, $%Hotkey_F10%, F10Action, On
GuiControl, Main:, BindStatus, F10 已绑定为：%n%
MsgBox, 262144,, F10（快速切断）已绑定为：%n%
return
BindF11:
GuiControlGet, n,, F11Edit
if (n = Hotkey_F11) {
  MsgBox, 262144,, 该键位已经是 F11 的键位，无需重复绑定！
  return
}
if (!IsKeyValid(n)) {
  MsgBox, 48,, 按键 "%n%" 无效！`n请输入正确的按键名称。
  return
}
if (n = Hotkey_F1 || n = Hotkey_F2 || n = Hotkey_F3 || n = Hotkey_F10 || n = Hotkey_F12) {
  MsgBox, 48,, 该键位已被其他控制键占用！
  return
}
try Hotkey, $%Hotkey_F11%, F11Action, Off
  Hotkey_F11 := n
Hotkey, $%Hotkey_F11%, F11Action, On
GuiControl, Main:, BindStatus, F11 已绑定为：%n%
MsgBox, 262144,, F11（三四五连卡）已绑定为：%n%
return
BindF12:
GuiControlGet, n,, F12Edit
if (n = Hotkey_F12) {
  MsgBox, 262144,, 该键位已经是 F12 的键位，无需重复绑定！
  return
}
if (!IsKeyValid(n)) {
  MsgBox, 48,, 按键 "%n%" 无效！`n请输入正确的按键名称。
  return
}
if (n = Hotkey_F1 || n = Hotkey_F2 || n = Hotkey_F3 || n = Hotkey_F10 || n = Hotkey_F11) {
  MsgBox, 48,, 该键位已被其他控制键占用！
  return
}
try Hotkey, $%Hotkey_F12%, F12Action, Off
  Hotkey_F12 := n
Hotkey, $%Hotkey_F12%, F12Action, On
GuiControl, Main:, BindStatus, F12 已绑定为：%n%
MsgBox, 262144,, F12（暴力出弯）已绑定为：%n%
return
```

## 三十一、功能启停切换函数

各功能独立的开/关切换：注册/注销热键、更新状态颜色与提示；包含连卡开启时默认回到四连等逻辑。

```ahk
ToggleF10:
ToggleCutFunc()
return
ToggleF11:
ToggleLianKaFunc(true)
return
ToggleF12:
ToggleBaoLiFunc()
return
ToggleBaoLi:
ToggleBaoLiFunc()
return
ToggleBaoLiFunc() {
  global
  baoLiOn := !baoLiOn
  if (baoLiOn) {
    Hotkey, $%Hotkey_BaoLi%, FuncWrapper, On
    SetStateColor("StateB", "已启用", "009933")
    Tooltip, 暴力出弯已开启
    GuiControl, Main:, BindStatus, 暴力出弯已开启
  } else {
    Hotkey, $%Hotkey_BaoLi%, FuncWrapper, Off
    SetStateColor("StateB", "未启用", "FF0000")
    Tooltip, 暴力出弯已关闭
    GuiControl, Main:, BindStatus, 暴力出弯已关闭
  }
  UpdateFKeyStates()
  SetTimer, RemoveTooltip, -1500
}
ToggleWuHen:
wuHenOn := !wuHenOn
if (wuHenOn) {
  Hotkey, $%Hotkey_WuHen%, FuncWrapper, On
  SetStateColor("StateW", "已启用", "009933")
} else {
  Hotkey, $%Hotkey_WuHen%, FuncWrapper, Off
  SetStateColor("StateW", "未启用", "FF0000")
}
return
ToggleSpace:
ToggleCutFunc()
return
ToggleCutFunc() {
  global
  spaceCut := !spaceCut
  if (spaceCut) {
    Hotkey, %Hotkey_Cut%, FuncWrapper, On
    SetStateColor("StateS", "已启用", "009933")
    Tooltip, 快速切断已开启
    GuiControl, Main:, BindStatus, 快速切断已开启
  } else {
    Hotkey, %Hotkey_Cut%, FuncWrapper, Off
    SetStateColor("StateS", "未启用", "FF0000")
    Tooltip, 快速切断已关闭
    GuiControl, Main:, BindStatus, 快速切断已关闭
  }
  UpdateFKeyStates()
  SetTimer, RemoveTooltip, -1500
}
ToggleKaQi1:
kaQi1On := !kaQi1On
if (kaQi1On) {
  if (Hotkey_KaQi1 = "Left" || Hotkey_KaQi1 = "Down") {
  } else {
    Hotkey, $%Hotkey_KaQi1%, FuncWrapper, On
  }
  SetStateColor("StateK1", "已启用", "009933")
} else {
  if (Hotkey_KaQi1 != "Left" && Hotkey_KaQi1 != "Down") {
    Hotkey, $%Hotkey_KaQi1%, FuncWrapper, Off
  }
  SetStateColor("StateK1", "未启用", "FF0000")
}
UpdateFKeyStates()
return
ToggleKaQi2:
kaQi2On := !kaQi2On
if (kaQi2On) {
  if (Hotkey_KaQi2 = "Left" || Hotkey_KaQi2 = "Down") {
  } else {
    Hotkey, $%Hotkey_KaQi2%, FuncWrapper, On
  }
  SetStateColor("StateK2", "已启用", "009933")
} else {
  if (Hotkey_KaQi2 != "Left" && Hotkey_KaQi2 != "Down") {
    Hotkey, $%Hotkey_KaQi2%, FuncWrapper, Off
  }
  SetStateColor("StateK2", "未启用", "FF0000")
}
UpdateFKeyStates()
return
ToggleLianKa:
ToggleLianKaFunc(false)
return
ToggleLianKaFunc(ResetToFour) {
  global
  lianKaOn := !lianKaOn
  if (lianKaOn) {
    if (ResetToFour) {
      LianKaCount := 4
      LianKaCountDesc := "四连"
      if (DelayWindowOpen)
        GuiControl, Delay:, DLK4, 1
      if (OverviewWindowOpen)
        RefreshOverview()
    }
    if (Hotkey_LianKa != "Right") {
      Hotkey, $%Hotkey_LianKa%, FuncWrapper, On
    }
    SetStateColor("StateLK", "已启用", "009933")
    Tooltip, 三四五连卡已开启
    ExtraHint := ResetToFour ? "（默认四连）" : ""
    GuiControl, Main:, BindStatus, 三四五连卡已开启%ExtraHint%
  } else {
    if (Hotkey_LianKa != "Right") {
      Hotkey, $%Hotkey_LianKa%, FuncWrapper, Off
    }
    SetStateColor("StateLK", "未启用", "FF0000")
    Tooltip, 三四五连卡已关闭
    GuiControl, Main:, BindStatus, 三四五连卡已关闭
  }
  SetTimer, RemoveTooltip, -1500
}
ToggleKaQiBothFunc() {
  global
  if (kaQi1On && kaQi2On) {
    kaQi1On := false
    kaQi2On := false
    if (Hotkey_KaQi1 != "Left" && Hotkey_KaQi1 != "Down") {
      Hotkey, $%Hotkey_KaQi1%, FuncWrapper, Off
    }
    if (Hotkey_KaQi2 != "Left" && Hotkey_KaQi2 != "Down") {
      Hotkey, $%Hotkey_KaQi2%, FuncWrapper, Off
    }
    SetStateColor("StateK1", "未启用", "FF0000")
    SetStateColor("StateK2", "未启用", "FF0000")
    Tooltip, 一二段连卡已关闭
    GuiControl, Main:, BindStatus, 一二段连卡已关闭
  } else {
    kaQi1On := true
    kaQi2On := true
    if (Hotkey_KaQi1 != "Left" && Hotkey_KaQi1 != "Down") {
      Hotkey, $%Hotkey_KaQi1%, FuncWrapper, On
    }
    if (Hotkey_KaQi2 != "Left" && Hotkey_KaQi2 != "Down") {
      Hotkey, $%Hotkey_KaQi2%, FuncWrapper, On
    }
    SetStateColor("StateK1", "已启用", "009933")
    SetStateColor("StateK2", "已启用", "009933")
    Tooltip, 一二段连卡已开启
    GuiControl, Main:, BindStatus, 一二段连卡已开启
  }
  UpdateFKeyStates()
  SetTimer, RemoveTooltip, -1500
}
```

## 三十二、退出与窗口关闭

退出前解绑全部热键、抬起所有按下的按键；主窗口关闭时走同一退出流程。

```ahk
ExitApp:
Hotkey, $%Hotkey_BaoLi%, FuncWrapper, Off
Hotkey, $%Hotkey_WuHen%, FuncWrapper, Off
Hotkey, %Hotkey_Cut%, FuncWrapper, Off
if (Hotkey_KaQi1 != "Left" && Hotkey_KaQi1 != "Down") {
  Hotkey, $%Hotkey_KaQi1%, FuncWrapper, Off
}
if (Hotkey_KaQi2 != "Left" && Hotkey_KaQi2 != "Down") {
  Hotkey, $%Hotkey_KaQi2%, FuncWrapper, Off
}
if (Hotkey_LianKa != "Right") {
  Hotkey, $%Hotkey_LianKa%, FuncWrapper, Off
}
Hotkey, $%Hotkey_F1%, F1Action, Off
Hotkey, $%Hotkey_F2%, F2Action, Off
Hotkey, $%Hotkey_F3%, F3Action, Off
Hotkey, $%Hotkey_F10%, F10Action, Off
Hotkey, $%Hotkey_F11%, F11Action, Off
Hotkey, $%Hotkey_F12%, F12Action, Off
Send, {%NitroKey% up}
Send, {%ForwardKey% up}
Send, {%BoostKey% up}
Send, {%DriftKey% up}
ExitApp
GuiClose:
Gosub, ExitApp
return
```

---

## 附：源码规模

- 原始总行数：**1646** 行
- 脚本类型：AutoHotkey v1（`.ahk`）
- 配置文件：脚本同目录下 `酒窝配置.ini`
