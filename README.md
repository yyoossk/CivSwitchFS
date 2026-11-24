# CivSwitchFS

## About this fork / このフォークについて

This is a fork of [SeanHeuc/CivSwitchFS](https://github.com/SeanHeuc/CivSwitchFS).
It carries seven improvements on top of upstream, added in three releases (improved-20261117, improved-20261121, improved-20261124).
Everything else is unchanged from upstream: the delay tables, the attempt budget (10, up to 30), and the wave budgets.

### improved-20261117

1. **Timing learning** (src/core/timing-slot.mjs, src/core.js, src/core/autoloader.mjs, persist/save-codec.js)
When the FS canary campaign succeeds, the winning timing schedule is recorded inside the save itself (a fixed-size record segment in `XSTRIG.Civ7Save`).
The next boot starts from the recorded schedule instead of the stock table.
Invalid records are ignored, and reinstalling the save resets the record, so the change is fully reversible.
On screen: `TIMING: recorded schedule loaded (win N); starting from it.`

2. **Freeze fix** (src/core/ipc/transport.js)
IPC sends were bare `svcSendSyncRequest` calls with no timeout, so an unresponsive file-system service froze the console (observed at `Checking the canary window...`).
Sends now run on a sacrificial thread with a 10-second bounded wait.
If the service stops replying, the run ends safely with `File-system service stopped responding; the run was stopped safely.`

Verified on real hardware (Switch 2, firmware 23.0.1, Japanese console and the Japanese version of Civilization VII).
With timing learning, the autoloader booted kaiHEN at a high rate from the second boot onward.

### improved-20261121 — retry reliability

The design of these three changes was ported from the slop branch of [s8n/CivSwitchFS-kaiHEN](https://github.com/s8n/CivSwitchFS-kaiHEN).

3. **Retirement precheck** (src/core/helpers/cache.mjs, src/core/steps/cleanup.mjs)
Before running the cache-retirement waves, the payload reads back the candidate cache entry and classifies it (original / aliased / unexpected / unavailable).
If the alias is already gone — a rejected timing window can still have cleared the valid byte — the retirement waves are skipped entirely and the state counts as verified.
Fewer runs end in a false `Cleanup failed; another attempt is unsafe.` stop, so more failures become safe retries within the same boot.

4. **Cleanup cache invalidate** (src/core/ipc/filesystem.js, src/core/steps/cleanup.mjs, src/core.js)
Before deleting the scratch tree and committing, cleanup invalidates the cached dirty scratch file (`save:/hole/big8`) via IFile::OperateRange.
The delete and commit therefore start from a clean cache state.

5. **Retune threshold** (src/core/steps/campaign.mjs)
The canary campaign now retunes its delays unless at least 80% of windows qualify (previously 50%).
In-run timing adjustments engage more readily across the attempt budget.

### improved-20261124

6. **OperateRange packet size corrected** (src/core/ipc/storage.js, src/core/ipc/filesystem.js)
The cache-invalidate commands (IStorage::OperateRange and IFile::OperateRange) declared 0x0C raw words for a 24-byte input.
The correct count is 0x0E: 0x10 bytes of padding + the 0x10-byte CMIF header + the 0x18-byte input, divided by 4.
This matches the libnx cmifMakeRequest formula and every other call site in the tree.
The old value caused no observed failure, because fsp-srv does not truncate the input struct at the declared count.
Both call sites now declare 0x0E.

7. **Dirty file size reduced** (src/core/config.mjs)
DIRTY_FILE_SIZE goes from 0x800000 (8 MiB) to 0x700000 (7 MiB).
This leaves 1 MiB of headroom under the BC save-data cap for metadata and other outstanding data.
The scratch tree is committed at setup time, so its metadata is flushed before the race begins.

Verification status: improvements 1 and 2 are verified on real hardware.
Improvements 3 to 7 are verified at the source and payload level only.

### このフォークの改良点

このフォークは [SeanHeuc/CivSwitchFS](https://github.com/SeanHeuc/CivSwitchFS) に、3回のリリース (improved-20261117、improved-20261121、improved-20261124) で7つの改良を加えたものです。
それ以外は upstream のままです。
遅延テーブル、アテンプト数 (10回、最大30回)、ウェーブ数は変更していません。

improved-20261117:

1. **タイミング学習** (src/core/timing-slot.mjs、src/core.js、src/core/autoloader.mjs、persist/save-codec.js)
FS の canary が成功したときのタイミングを、セーブ内 (XSTRIG.Civ7Save の固定長レコード) に記録します。
次回起動時は初期テーブルではなく、記録したタイミングから開始します。
記録が不正なら無視され、セーブを入れ直せばリセットされる可逆な実装です。
画面表示: `TIMING: recorded schedule loaded (win N); starting from it.`

2. **フリーズ修正** (src/core/ipc/transport.js)
IPC 送信はタイムアウトのない `svcSendSyncRequest` で、FS が無応答になると本体が凍結していました (「Checking the canary window...」で停止)。
送信を使い捨てスレッドで実行し、最大10秒で打ち切るようにしました。
無応答時は固まらずに `File-system service stopped responding; the run was stopped safely.` と表示して安全に終了します。

実機確認済みです (Switch 2 / FW 23.0.1 / 国内版本体・日本版シヴィライゼーション7)。
タイミング学習により、オートローダーは2回目以降に高い確率で kaiHEN を起動しました。

improved-20261121 (リトライの信頼性。設計は [s8n/CivSwitchFS-kaiHEN](https://github.com/s8n/CivSwitchFS-kaiHEN) の slop ブランチから移植):

3. **リタイア前チェック** (src/core/helpers/cache.mjs、src/core/steps/cleanup.mjs)
キャッシュのリタイア用ウェーブを走らせる前に、候補エントリを読み戻して状態を判定します (original / aliased / unexpected / unavailable)。
エイリアスが既に消えていればウェーブ自体を省略し、安全とみなします。
タイミングが外れた場合でも有効バイトが消えていることがあるためです。
「Cleanup failed; another attempt is unsafe.」で止まってしまうケースが減り、同一ブート内で安全に再試行できる幅が広がります。

4. **クリーンアップ時のキャッシュ無効化** (src/core/ipc/filesystem.js、src/core/steps/cleanup.mjs、src/core.js)
スクラッチを削除してコミットする前に、汚れたスクラッチファイル (`save:/hole/big8`) のキャッシュを IFile::OperateRange で無効化します。
削除とコミットがクリーンなキャッシュ状態から始まります。

5. **再調整のしきい値** (src/core/steps/campaign.mjs)
タイミング再調整が働く条件を「合格率50%未満」から「80%未満」に変更しました。
アテンプトの中で調整が効きやすくなります。

improved-20261124:

6. **OperateRange のパケットサイズを修正** (src/core/ipc/storage.js、src/core/ipc/filesystem.js)
キャッシュ無効化の命令 (IStorage::OperateRange と IFile::OperateRange) が、24バイトの入力に対して 0x0C 語と宣言していました。
正しいのは 0x0E 語です。
パディング 0x10 バイト + CMIF ヘッダ 0x10 バイト + 入力 0x18 バイトを4で割った値です。
libnx の cmifMakeRequest の計算式と、ツリー内の他の全ての呼び出しと一致します。
古い値でも失敗した形跡はありません。
本体側が宣言された語数で入力を切り捨てていないためです。
両方の呼び出しを 0x0E に揃えました。

7. **ダーティファイルのサイズを縮小** (src/core/config.mjs)
DIRTY_FILE_SIZE を 0x800000 (8 MiB) から 0x700000 (7 MiB) にしました。
BC セーブデータの上限に対して、メタデータなどの分として 1 MiB の余裕を残します。
スクラッチツリーはセットアップ時に commit されるため、メタデータはレース開始前にフラッシュされます。

検証の状態: 改良1と2は実機確認済みです。
改良3から7は、ソースとペイロードのレベルでの検証のみです。

---

> [!IMPORTANT]
> Thanks to [**Slapah**](https://github.com/byizmysh/IZCIVSwitch/blob/main/NOTICE.md#upstream) for the ns2jb filesystem port to CivSwitch and [**Kurumuz**](https://gitlab.com/kurumuz/ns2jb) for the improved fork of ns2jb from [**pepsipu**](https://gitlab.com/pepsipu/ns2jb), which laid the groundwork for this research. All kudos goes to them for the Civ FS port :heart:

This is payload provides **access to FS privileges** through an elevated fsp-srv handle. It runs inside the **CivSwitch Remote JS Loader**. That elevated session is exposed as `globalThis.fspSrv` for the next payloads.

## A bit of a preface

CivSwitchFS is **semi-untethered**, RPC was fully removed :tada: ! It should also be more consistent for any hardware.

**It's intended for devs**, as all I give is a god `fsp-srv` handle (full FS) for you to play with. The rest is for you to figure out.

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
using your usual CivSwitch setup. Then send the payload 

~~~sh
npm run send -- <console-ip>
~~~

Wait for **`You just Civ Sevened your machine, congrats`** which obviously means success :tada: !
Then, the loader automatically shows `Awaiting payload at <ip>:9026` and you are free to send payloads.

### Booting faster next time (this will be automatised eventually)

After success, the payload prints the working `roleDelays` and `guardDelays`
values in nanoseconds. To use them as the starting schedule next time, replace
the arrays in [`src/core/config.mjs`](src/core/config.mjs#L44-L47):

~~~js
var roleDelays = [0, 300000, 350000, 450000, 550000, 650000, 1000000, 1200000, 1300000, 1700000];
var guardDelays = [150000, 300000];
~~~

The trigger's initial sleep is 1,600,000 ns. The [Kurumuz](https://gitlab.com/kurumuz/ns2jb)
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
- **Slapah/CivSwitch**: first true working ns2jb port for Civ. Sadly after his code beeing stole, he chose to nuke it.
- **[Kurumuz/ns2jb](https://gitlab.com/kurumuz/ns2jb)**: improved version of ns2jb


## License

The original code in this repository is distributed under the MIT license. See [LICENSE](LICENSE) and [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) for detailed notices and credits.