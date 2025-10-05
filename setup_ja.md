# Arch Linux インストールガイド

<br />

## 0. ISO イメージを作成する

それでは早速インストールを始めましょう。  
まずは、公式の [Arch Linux ダウンロードページ](https://archlinux.org/download/) から **Arch Linux のインストールイメージ**をダウンロードします。  
ファイル名は次のような形式になっています。

```

archlinux-[日付]-x86_64.iso

````

ダウンロードが完了したら、ISOをUSBメモリに書き込みます。  
（例：Rufus・balenaEtcher・`dd` コマンドなどを使用）

<br />

## 1. ディスクを初期化して新しいGPTパーティションを作成する

```bash
sudo gdisk /dev/nvme0n1
````

`gdisk` 内で以下のように入力します。

```
o   # 新しいGPTパーティションテーブルを作成
y   # 確認して既存のテーブルを削除

--- 

n   # 新しいパーティション → ルート用
1   # パーティション番号
    # 先頭セクタはデフォルト
+100G  # サイズ = 100GB
8304   # Linux x86-64ルート用のタイプコード（またはEnterでOK）

---

n   # 新しいパーティション → スワップ領域
2   # パーティション番号
    # 先頭セクタはデフォルト
+16G  # サイズ = 16GB（必要に応じて変更）
8200  # Linux swap のタイプコード

---

n   # 新しいパーティション → EFIシステム
3   # パーティション番号
    # 先頭セクタはデフォルト
+512M  # サイズ = 512MB
ef00   # EFI System Partition 用タイプコード

---

w   # 変更を保存
y   # 確認
```

✅ 結果として以下のようなパーティション構成になります：

| パーティション | サイズ    | タイプ              | マウントポイント |
| :------ | :----- | :--------------- | :------- |
| p1      | 100 GB | Linux filesystem | `/`      |
| p2      | 16 GB  | Linux swap       | swap     |
| p3      | 512 MB | EFI System       | `/boot`  |

<br />

## 2. パーティションをフォーマットする

```bash
sudo mkfs.ext4 /dev/nvme0n1p1
```

→ ルートパーティションを `ext4` 形式にフォーマット。

```bash
sudo mkswap /dev/nvme0n1p2
```

→ スワップ領域を初期化。

```bash
sudo mkfs.fat -F32 /dev/nvme0n1p3
```

→ EFIパーティションをFAT32でフォーマット（UEFIに必要）。

<br />

## 3. パーティションをマウントする

```bash
sudo mount /dev/nvme0n1p1 /mnt
```

→ ルートパーティションを `/mnt` にマウント。

```bash
sudo mkdir /mnt/boot
sudo mount /dev/nvme0n1p3 /mnt/boot
```

→ EFIパーティションを `/boot` にマウント。

```bash
sudo swapon /dev/nvme0n1p2
```

→ スワップを有効化。

<br />

## 4. 基本システムをインストールする

```bash
sudo pacstrap -K /mnt base linux linux-firmware nano networkmanager grub efibootmgr
```

→ 基本パッケージ・Linuxカーネル・ファームウェア・テキストエディタ・ネットワーク・ブートローダ関連をまとめてインストール。

<br />

## 5. `fstab` を生成する

```bash
sudo genfstab -U /mnt | sudo tee /mnt/etc/fstab
```

→ パーティション構成を自動検出し、`/etc/fstab` にマウント情報を出力。

<br />

## 6. 新しいシステムに入る（chroot）

```bash
sudo arch-chroot /mnt
```

→ インストールしたシステム環境に入ります（この状態で設定を行います）。

<br />

## 7. タイムゾーン・ロケール・ホスト名を設定

### タイムゾーンを設定

```bash
ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
hwclock --systohc
```

→ 日本時間に設定。

### ロケール（言語設定）を生成

```bash
nano /etc/locale.gen
```

以下の行のコメントを外します：

```
en_US.UTF-8 UTF-8
ja_JP.UTF-8 UTF-8
```

その後：

```bash
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

→ システムの言語を設定。

### ホスト名とホスト情報を設定

```bash
echo "arch-pc" > /etc/hostname
nano /etc/hosts
```

以下を追加：

```
127.0.0.1   localhost
::1         localhost
127.0.1.1   arch-pc.localdomain   arch-pc
```

<br />

## 8. ユーザー作成とsudo権限付与

```bash
useradd -m -G wheel -s /bin/bash ruki
passwd ruki
```

→ メインユーザーを作成し、`wheel` グループに追加。

sudo設定を有効化：

```bash
EDITOR=nano visudo
```

以下の行のコメントを外します：

```
%wheel ALL=(ALL) ALL
```

<br />

## 9. ネットワークと時刻同期を有効化

```bash
systemctl enable NetworkManager
systemctl enable systemd-timesyncd
```

→ 起動時に自動的にインターネット接続＆時刻同期を有効に。

<br />

## 10. GRUBブートローダー（UEFI）をインストール

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub
grub-mkconfig -o /boot/grub/grub.cfg
```

→ GRUBをEFIパーティションにインストールし、ブートメニューを生成。

<br />

## 11. 最後の仕上げ

```bash
exit
umount -R /mnt
swapoff -a
reboot
```

→ chrootを抜け、アンマウント＆スワップ解除して再起動。

<br />

✅ 再起動後：

* ユーザー `ruki` でログイン
* インターネット接続は自動で有効
* `sudo` コマンドで管理操作可能
* 時刻とロケールも正しく設定済み

---

## やりたい人向け：便利ツールのインストール

```bash
sudo pacman -Syu git base-devel vim htop man-db wget curl
```

→ パッケージ更新と基本ユーティリティの導入。

---

### 参考資料

* [Arch Wiki: Installation Guide](https://wiki.archlinux.org/title/Installation_guide)
* [GRUB UEFI Setup](https://wiki.archlinux.org/title/GRUB)
