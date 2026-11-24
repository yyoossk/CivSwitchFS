# CivSwitchFS

## About this fork / このフォークについて

This is a fork of [SeanHeuc/CivSwitchFS](https://github.com/SeanHeuc/CivSwitchFS).

The current release, stock-20261125b, is the stock upstream payload plus three fixes.

Everything else is unchanged from upstream: the delay tables, the attempt budget (10, up to 30), and the wave budgets.

### The three fixes

1. **OperateRange packet size corrected** (src/core/ipc/storage.js)

The cache-invalidate command (IStorage::OperateRange) declared 0x0C raw words for a 24-byte input.

The correct count is 0x0E: 0x10 bytes of padding + the 0x10-byte CMIF header + the 0x18-byte input, divided by 4.

This matches the libnx cmifMakeRequest formula and every other call site in the tree.

The old value caused no observed failure, because fsp-srv does not truncate the input struct at the declared count.

2. **Dirty file size reduced** (src/core/config.mjs)

DIRTY_FILE_SIZE goes from 0x800000 (8 MiB) to 0x700000 (7 MiB).

This leaves 1 MiB of headroom under the BC save-data cap for metadata and other outstanding data.

3. **Installer creates the save folders** (persist/installer.js)

When the console's save data has no saves/Single folder, installing the autoloader save failed at the file creation step with `create : 0x202` (PathNotFound).

The installer now creates `save:/saves` and `save:/saves/Single` first when they are missing.

Creating them is a no-op when they already exist.

### Why the current release is nearly stock

Earlier releases of this fork (improved-20261117, improved-20261121, improved-20261124) carried more improvements: timing learning, a freeze fix, and retry changes.

On the test console, those payloads stopped at the first step of every boot: "Could not open the required file systems."

The stock payload boots on the same console, and stock-20261125b booted on the first try.

Which of the earlier changes breaks the first step is still under investigation.

Until that is known, the current release keeps only the three fixes above.

Verified on real hardware (Switch 2, firmware 23.0.1, Japanese console and the Japanese version of Civilization VII): stock-20261125b booted kaiHEN on the first try.

### このフォークの改良点

このフォークは [SeanHeuc/CivSwitchFS](https://github.com/SeanHeuc/CivSwitchFS) のフォークです。

現在のリリース stock-20261125b は、純正の upstream ペイロードに修正を3点加えたものです。

それ以外は upstream のままです。

遅延テーブル、アテンプト数 (10回、最大30回)、ウェーブ数は変更していません。

修正は次の3点です。

1. **OperateRange のパケットサイズを修正** (src/core/ipc/storage.js)

キャッシュ無効化の命令 (IStorage::OperateRange) が、24バイトの入力に対して 0x0C 語と宣言していました。

正しいのは 0x0E 語です (padding 0x10 バイト + CMIF ヘッダ 0x10 バイト + 入力 0x18 バイトを4で割ったもの)。

libnx の cmifMakeRequest の公式と、このツリーの他の全ての呼び出しと一致します。

古い値でも実害は観測されていません。

fsp-srv が宣言された語数で入力構造体を切り捨てていないためです。

2. **ダーティファイルを小さく** (src/core/config.mjs)

DIRTY_FILE_SIZE を 0x800000 (8MiB) から 0x700000 (7MiB) にしました。

BC のセーブデータ上限に対して、メタデータや他の未保存データ用に 1MiB の余裕を残します。

3. **インストーラーがセーブのフォルダを作成** (persist/installer.js)

本体のセーブデータに saves/Single フォルダが無いと、オートローダーのセーブを入れる際にファイル作成で `create : 0x202` (PathNotFound) になりました。

インストーラーが、無い場合に `save:/saves` と `save:/saves/Single` を先に作るようになりました。

既にある場合は何もしません。

### 現在のリリースがほぼ純正な理由

このフォークの以前のリリース (improved-20261117、improved-20261121、improved-20261124) には、タイミング学習、フリーズ修正、リトライ変更など、もっと多くの改良が入っていました。

テスト機では、それらのペイロードが毎回起動の最初の段階で止まりました (「Could not open the required file systems.」)。

同じ本体で純正ペイロードは起動し、stock-20261125b は一発で起動しました。

以前の変更のどれが最初の段階を止めているかは調査中です。

それが分かるまでは、現在のリリースには上記の3点だけを入れています。

実機確認済みです (Switch 2、ファームウェア 23.0.1、日本版本体と日本版シヴィライゼーション7)。

stock-20261125b で kaiHEN が一発で起動しました。

## A bit of a preface

CivSwitchFS is **semi-untethered**, RPC was fully removed :tada: ! It should also be more consistent for any hardware.

**It's intended for devs**, as all I give is a god `fsp-srv` handle (full FS) for you to play with.

The rest is for you to figure out.

> [!NOTE]
> If you are looking for a **reliable KaiHEN access**, this [fork](https://github.com/s8n/CivSwitchFS/tree/slop) from [@Ave](https://github.com/aveao) will get you there in no time.

### Tested configuration

- Nintendo Switch 2, firmware **23.0.1**
- **Civilization VII 1.0.0**, physical cartridge
- CivSwitch Remote JS Loader already running

### Requirements

- Node.js **20 or later** and npm
- A Switch 2 with the CivSwitch Remote JS Loader already running and reachable

  on the local network when installing persistence or sending a payload

## Usage

Install dependencies once from this repository:

~~~sh

npm install

~~~

### Persist the payload (optionnal)

With the CivSwitch Remote JS Loader running, install the persistent image:

~~~sh

npm run persist:install -- <console-ip> [--minify-payload]

~~~

Future launches from that save start the exploit, loader, and

payload automatically, without a computer or network connection.

The installer does not back up the old save.

### Launch the exploit

If you are not going for persistence, just start the CivSwitch Remote JS Loader

using your usual CivSwitch setup.

Then send the payload 

~~~sh

npm run send -- <console-ip>

~~~

Wait for **`You just Civ Sevened your machine, congrats`** which obviously means success :tada: !

Then, the loader automatically shows `Awaiting payload at <ip>:9026` and you are free to send payloads.

### Booting faster next time (this will be automatised eventually)

After success, the payload prints the working `roleDelays` and `guardDelays`

values in nanoseconds.

To use them as the starting schedule next time, replace

the arrays in [`src/core/config.mjs`](src/core/config.mjs#L44-L47):

~~~js

var roleDelays = [0, 300000, 350000, 450000, 550000, 650000, 1000000, 1200000, 1300000, 1700000];

var guardDelays = [150000, 300000];

~~~

The trigger's initial sleep is 1,600,000 ns.

The [Kurumuz](https://gitlab.com/kurumuz/ns2jb)

reference schedule used these instead (might help if it worked for you on those)

~~~js

var roleDelays = [0, 300000, 350000, 450000, 550000, 650000, 1700000, 2000000, 2150000, 2800000];

var guardDelays = [0, 150000];

~~~

If you use the persisted payload, you ofc need to rerun the installer so its saved image includes the new timings.

> [!NOTE]
> You can play with the delays and MORE in `src/core/config.mjs`, try some shit (but don't brick pls) !

## Example FS usage

After the main payload reports success, you can send follow-up payloads and use

the `globalThis.fspSrv` handle to perform filesystem operations.

An example can be found in `examples/user-storage`.

## Notes

- The auto timing calibration is not perfect yet, some freezes can happen
- Timing configuration, retries and shit will go to a persisted config file soon enough
- THERE IS SO MUCH MORE to be done, I'm just releasing now so you guys can study the semi-untethered stuff
- You can learn about how the exploit works [there](https://github.com/ItsNerisma/CivSwitchFS/blob/main/HOW_IT_WORKS.md)

## Project structure

```text
src/core.js          # payload entry point and flow
src/core/            # steps, memory access, and FS communication
src/core/config.mjs  # the whole exploit conf, tweak it if you want
examples/            # small payload usage examples
persist/             # persistence builder inputs, save codec, writer, and installer
tools/               # well, tools
```

## Credits

- **[Gezine/CivSwitch](https://github.com/Gezine/CivSwitch)**: userland exploit and persistent installer
- **[pepsipu/ns2jb](https://gitlab.com/pepsipu/ns2jb)**: og FS exploit
- **Slapah/CivSwitch**: first true working ns2jb port for Civ.

  Sadly after his code beeing stole, he chose to nuke it.
- **[Kurumuz/ns2jb](https://gitlab.com/kurumuz/ns2jb)**: improved version of ns2jb


## License

The original code in this repository is distributed under the MIT license.

See [LICENSE](LICENSE) and [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) for detailed notices and credits.