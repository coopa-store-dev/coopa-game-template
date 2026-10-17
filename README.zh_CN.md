<p align="right">
  <a href="README.md">English</a> · <strong>简体中文</strong>
</p>

# Coopa 游戏模板

用这个模板给 Coopa 卡做游戏。里面是一个能玩的小游戏「拍拖鞋」:库巴从三个洞里冒出来,▲▼ 移动拖鞋,● 拍它。

## 开始

1. 在 GitHub 上点 **Use this template**,建一个你自己的仓库。
2. 在你的仓库里点 **Code → Codespaces → Create codespace**。第一次打开要等几分钟(在装 SDK)。
3. 给游戏起个 id(小写字母开头,小写字母和数字,最多 15 个),在终端里运行 `coopa rename 你的id`。
4. 运行 `coopa run`:浏览器会打开一个新标签,里面就是卡的画面。↑↓ 是 ▲▼,空格是 ●。
5. 改代码,再 `coopa run` 看效果。
6. 运行 `coopa check`:跑一遍和商店一样的检查(开机、截图、乱按 5 分钟、存档大小)。

每次推送,「自查」流程(Actions)也会自动跑一遍,还会编卡上的固件、查游戏大小。

也可以在自己电脑上用 VS Code 的 Dev Containers 打开(要装 Docker)。

SDK 出了新版(`coopa version` 看装的是第几版,最新的在 [coopa-sdk 的 Releases](https://github.com/coopa-store-dev/coopa-sdk/releases)):
在命令面板里运行 **Codespaces: Rebuild Container**(自己电脑上是 **Dev Containers: Rebuild Container**),会重新装最新的 SDK。

## 文件夹里有什么

| 文件 | 是什么 |
|---|---|
| `coopa.toml` | 游戏的「身份证」:id、名字、版本、SDK 版本、入口、里程碑 |
| `<id>_texts.h` | 屏幕上的全部文字(字库从这里收集字) |
| `src/<id>_logic.*` | 玩法:纯 C,不碰屏幕和存档 |
| `src/<id>_save.*` | 存档格式和里程碑 |
| `src/<id>_ui.*` | 画面(LVGL 和 `ui_kit.h`) |
| `src/<id>_app.c` | 入口表 `kit_app_t`:卡的外壳通过它叫你的游戏 |

## 规则(不守规则不能上架)

1. **id**:小写字母开头,小写字母和数字,最多 15 个。它也是你的存档名字。
2. **名字**:不是 `static` 的函数和全局变量,名字都以 `你的id_` 开头(上百个游戏编进同一份固件,重名就编不过)。
3. **存档**:只用 `kit_sv_blob_load` / `kit_sv_blob_store` / `kit_sv_ns_erase`,命名空间就是你的 id,最多 2 KB。
4. **文字**:屏幕上的字全写在 `<id>_texts.h` 里,`.c` 里不写中文。字体里没有的字会报错(卡上会显示成空白)。
5. **不许用**:联网、直接用蓝牙(两张卡联机用 `kit_link.h`,见下面)、直接读写 NVS / Flash / 分区、重启、睡眠、自己开任务、`EM_ASM` 等嵌 JavaScript、汇编。只用 SDK 给的 `kit_*.h`、`ui_kit.h`、`lvgl.h` 和 C 标准库。
6. **随机数**:在开局时取(`esp_random()`),不在 `init`、`milestones()`、`won()`、`status()` 里取。
7. **里程碑**:最多 8 个,在 `coopa.toml` 里按顺序写,只能往后加。
8. **大小**:游戏编成的游戏块最多 512 KB(`coopa check --card` 的「游戏块」一步会量)。

## 两张卡联机(外壳接口 2)

两个人各拿一张卡面对面玩,用 `kit_link.h`(`coopa.toml` 要写 `kit_api = 2`)。卡上走蓝牙,外壳管找人、连接、重发、断线;
你只收发最长 64 字节的消息:

```c
// 进联机画面:两边都调,不分主机;频道用你的游戏编号
kit_link_start(MYGAME_NO);

// frame() 里每帧看一眼
switch (kit_link_state()) {
case KIT_LINK_SEARCHING: /* 显示「正在找对方……」 */ break;
case KIT_LINK_NO_MEMORY: /* 显示「内存不够,先退出再进来」 */ break;
case KIT_LINK_LOST:      /* 显示「对方走开了,等他回来」 */ break;
case KIT_LINK_CONNECTED:
    if (kit_link_session() != s_seen_session) {  // 新的一次连接:先互相对一下进度
        s_seen_session = kit_link_session();
        send_hello();
    }
    uint8_t msg[KIT_LINK_MSG_MAX];
    size_t n;
    while ((n = kit_link_recv(msg, sizeof(msg))) > 0) handle(msg, n);
    break;
default: break;
}
```

- 一次连接之内消息不丢、不乱序;**断线时没送到的都丢了**,连回来要自己对账(比如互发「我走到第几步」)。
- `kit_link_send` 返回 `false`(还没连上,或排队满了 8 条)就是没发出去:留着,下一帧再发。
- 谁先手可以用 `kit_link_initiator()`:两边结果一定相反。
- 连上以后外壳不熄屏(还在找的时候照常熄屏);离开游戏时外壳替你关。一时没空取消息没关系,对方会等着,不会断。卡上开着蓝牙多占约 68 KB 内存,游戏自己的内存要省着用。
- 速度:一条消息来回约 60 ms(每 16 字节一片),适合回合制,不适合每帧同步画面。
- 在浏览器里测:`coopa run` 后打开它打印的 `duo.html`,一页两张卡,点哪张键盘就控制哪张;「断开 3 秒」测断线。

## 提交

Coopa 官网开放以后:用 GitHub 登录、申请成为开发者,批准后在官网提交你的仓库和 commit。
