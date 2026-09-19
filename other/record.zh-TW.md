# 除錯紀錄（繁體中文版）：Strict Confinement 的限制與 udev 權限設定

> 本文為 `record.md` 中兩個章節的繁體中文摘要／翻譯，方便日後快速查閱：
> - `Can strict confinement reach a host "deb GPU" without a content-interface snap?`
> - `udev permission configuration for GPU device nodes (e.g. /dev/galcore)`
>
> 完整英文原始紀錄（含所有機器的完整除錯過程）請見 `record.md`。

## 問題背景

在 NXP Leuven 這台機器上，我們已經成功做出一個 content-interface 的
provider snap（`imx8mp-gpu-drivers-core24/`），把 host 上用 apt 安裝的
Vivante GPU 驅動（`imx-gpu-viv`）透過 content interface 分享給
`opencl-cts` snap，讓 confined 狀態下的 `opencl-cts.clinfo` 也能抓到
"Vivante OpenCL Platform"。

但使用者提出：**這代表每一台「deb GPU」機器都要額外手工包一個 GPU
driver 的 snap 才行嗎？如果不想這樣做，能不能在維持 strict confinement
（不用 `--no-confinement`）的前提下，讓 `opencl-cts.clinfo` 依然能驗證
host 上用 deb 安裝的 GPU？** 另外也問到 udev 權限該怎麼正確設定。

---

## 結論一：Strict Confinement 底下，不透過 content-interface snap 是否能存取 host 的 deb GPU？

**實測結果：不行。**

### 驗證方式

在 G700 1 上做了一個一次性的測試用 snap（`sftest`），刻意宣告一個
`system-files` 的 plug：

```yaml
plugs:
  host-opencl:
    interface: system-files
    read:
      - /usr/lib/aarch64-linux-gnu
      - /etc/OpenCL
```

用 `--dangerous` 側載安裝後：

- `snap connect sftest:host-opencl` **竟然成功**了——因為
  `system-files` 這個 interface 雖然在 base declaration 裡寫了
  `allow-installation: false`（代表要上架 Snap Store 需要人工審核），
  但**本機側載（`--dangerous`）的 snap 不受這條規則阻擋**，所以
  connect 動作本身沒有被拒絕。
- 但實際在 confined 環境裡去讀取
  `/usr/lib/aarch64-linux-gnu/libOpenCL.so*` 時，**失敗**了
  （`No such file or directory`）。
- 用 `snap run --shell` 進到這個 snap 內部去看
  `/usr/lib/aarch64-linux-gnu`，發現看到的其實是 **base snap
  （core24）自己的 `/usr`**（裡面是 core24 內建的 cryptsetup、
  dhcpcd 等等），根本不是 host 真正的內容。
- 對照組：`/etc/OpenCL/vendors` 這個路徑，**即使完全沒有額外
  interface，也是可以直接穿透看到 host 內容的**（confined app 內可以
  看到 host 真正的 `/etc/OpenCL/vendors/libmali.icd` 符號連結）。但這個
  連結指向的目標
  `/usr/lib/aarch64-linux-gnu/mt8188/OpenCL/libmali.icd`，一樣會被解析
  到 base snap 自己的 `/usr`，所以就算看得到連結本身，也讀不到真正的
  driver 檔案內容，沒有用。

### 原因

Strict confinement 下，`/usr`、`/lib`、`/bin` 這幾個路徑，是被
**base snap（例如 core24）自己的內容整個取代掉**的——這是 snap
沙盒機制的基本設計。而 `/etc`、`/var`、`/home`、`/run` 這類路徑，則是
預設（或透過 `system-files`／`personal-files` interface）可以穿透看到
host 真正內容的。

換句話說：**沒有任何 snapd 的 interface，可以把 host 上任意
`/usr/lib/*` 底下的驅動檔案，暴露給一個 strict confinement 的 snap
看到。**

### 唯三可行的方法

若真的要讓一個 confined 的行程拿到「真正的」GPU driver 檔案內容
（不是空殼路徑），只有以下三條路：

1. **Content interface**（就是我們已經在 NXP Leuven 上做的方法）
   —— provider snap 分享一個路徑（例如 `$SNAP/graphics`），consumer
   snap 用對應的 plug 去掛載進來。**這是唯一「乾淨」、snapd 原生支援
   的方法**。只要同時要「strict confinement」+「host 上真正的驅動檔
   案」，這條路無法避免。
2. **Classic confinement** —— 直接放棄整個沙盒隔離機制（等於完全信任
   這個 snap，能看到整個 host 檔案系統），要上架 Snap Store 也需要人
   工審核。除非真的不在乎沙盒保護，否則不建議只為了這個目的採用。
3. **`--no-confinement`**（目前 G700 1 / CIX P1 採用的作法）——這其實
   不是走 snapd 的 interface 機制，而是直接繞過 `snap run`，把
   wrapper script 當成一般行程來執行，因此自然會繼承 host 真正的檔案
   系統路徑。但這正是使用者想要避免的做法。

### 實務上的結論

**對於沒有另外包 content-interface snap 的「deb GPU」機器**
（例如目前的 G700 1、CIX P1），`--no-confinement`（或 classic
confinement）**依然是唯一能讓 `opencl-cts.clinfo` 抓到 host 驅動的方
法**，沒辦法單純靠 interface 連線的方式做到。

如果要減少每次手工包一個 content-interface snap 的工作量，可以考慮把
NXP Leuven 這次用的做法（掃描常見 host 路徑找出
`libOpenCL.so*`／有 ICD symbol 的檔案、複製進 `graphics/lib` part、
沿用一樣的 `gpu-2404`／`gpu-2604` 風格 slot）寫成一個小型的自動化腳
本，日後遇到新機器時可以半自動產生對應的 content snap——但**「需要包
一個 snap」這件事本身無法被完全省略**，除非願意接受 classic
confinement 或 `--no-confinement`。

---

## 結論二：udev 權限該如何正確設定（以 `/dev/galcore` 為例）

有些 vendor 的 GPU device node，預設就是 **root-only**
（`crw-------  1 root root`），這會同時擋住：

- host 上一般使用者直接執行的 `clinfo`
- confined 狀態下的 `opencl-cts.clinfo`

兩者失敗訊息完全一樣（`Failed to open device: No such file or
directory`），用 `sudo` 執行則兩者都會成功。**這其實跟 snap
confinement 完全無關，是單純的 Linux device node 權限問題**，修法跟一
般 host 上的應用程式要解決這個問題時完全一樣。

### 解法：新增一條 udev 規則，把 group 權限開給非 root 的群組

參考同一台機器上 DRM 子系統既有的慣例（`/dev/dri/renderD128` 是
`render` 群組、`/dev/dri/card0` 是 `video` 群組），新增：

```
# /etc/udev/rules.d/99-galcore.rules
KERNEL=="galcore", MODE="0660", GROUP="video"
```

套用（不需要重開機）：

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger --name-match=galcore
```

再確認要執行的使用者有在這個群組裡（沒有的話要加，加完要重新登入才
會生效）：

```bash
sudo usermod -aG video "$USER"
```

### 已在 NXP Leuven 上實測驗證

套用規則後，`/dev/galcore` 的權限從 `crw------- root root` 變成
`crw-rw---- root video`。之後：

- host 上一般使用者身分執行 `clinfo`（apt 安裝的版本）
- confined 狀態下的 `opencl-cts.clinfo`（透過
  `imx8mp-gpu-drivers-core24` 這個 content-interface snap）

**兩者都不需要 `sudo`、也不需要 `--no-confinement`**，就能正確找到
"Vivante OpenCL Platform"。

### 其他 vendor device node 的通用作法

1. `ls -l /dev/<device>` 先看目前的擁有者跟權限模式。
2. 挑一個符合該平台慣例的群組（GPU/DRM 類裝置最常見的是 `video` 或
   `render` 這兩個）。
3. 寫一條 `KERNEL=="<name>", MODE="0660", GROUP="<group>"` 的規則，
   放到 `/etc/udev/rules.d/` 底下。
4. `udevadm control --reload-rules` + `udevadm trigger` 套用。
5. 確認使用者有在對應群組裡。

**特別注意**：snapd 自己會針對這類裝置自動產生一份
`70-snap.<snap名稱>.rules`（在 NXP Leuven 上已經觀察到這份規則存
在）——但那份規則**只是把裝置標記進該 snap 的 device cgroup
白名單**，讓 snapd 的沙盒機制「允許」這個 snap 存取這個裝置類別，
**並不會去改動裝置節點本身的檔案權限（DAC）**。所以就算 snapd 那份
規則存在，只要裝置節點本身還是 `root-only`，一般使用者身分還是會被
Linux 核心的權限檢查擋下來——這也是為什麼上面這條額外的 udev 規則仍
然是必要的，兩者是互補、而非互斥的關係。

---

## 相關檔案

- `record.md` — 完整英文版除錯紀錄（本文件為其兩個章節的中文摘要）。
- `imx8mp-gpu-drivers-core24/99-galcore.rules` — 上面提到的 udev 規則
  檔案本體，已加入版本控制方便直接複製到其他機器使用。
- `imx8mp-gpu-drivers-core24/README.md` — NXP Leuven content-interface
  snap 的建置／安裝／連線步驟，以及 `/dev/galcore` 權限問題的說明
  （英文）。
