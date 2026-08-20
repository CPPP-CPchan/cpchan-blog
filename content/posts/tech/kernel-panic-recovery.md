---
title: apt upgrade後にkernel panicで起動不能になったLinux機をLive USBで復旧した話
date: 2026-08-14T19:00:00+09:00
draft: false
tags:
  - Linux
  - トラブルシューティング
  - GRUB
  - DKMS
  - initramfs
---

## 概要

しぴちゃんには仕事で管理しているAI用サーバー (DGX OS)があります。サーバー機で `apt update && apt upgrade` を実行したところ、コマンド自体が異常終了。そのまま再起動をかけたら次のようなカーネルパニックでクラッシュループに陥り、起動不可になりました。

```
VFS: Unable to mount root fs on unknown-block(0,0)
```

原因の切り分けから、Live USB経由でのchroot復旧までの一連の対応を記録しておきましょう。同じ症状に遭遇した人の参考になれば幸いです。

なお、このトラブルシューティングはClaude Code (Sonnet 5)と共同で行い、解決後の記事執筆はClaude Sonnet 5で行っています。

## 症状

- `apt upgrade` でカーネルが新バージョンに更新された
- アップグレード処理の途中で失敗し、コマンドが異常終了
- そのまま再起動をかけたところ、GRUBの先でカーネルパニックが発生し起動不能に
- ヘッドレス運用のためGRUBのタイムアウトが `0` に設定されており、通常の起動シーケンスでは古いカーネルへのフォールバックができない状態だった

## 原因

`unable to mount root fs on unknown-block(0,0)` は、カーネルがroot filesystemをマウントするための initramfs（初期RAMディスク）を見つけられない場合に発生する典型的なエラーである。

今回のケースでは以下が重なっていた。

1. 新カーネルのイメージ (`vmlinuz-*`) はインストールされたが、対応する `initrd.img-*` が生成されないままアップグレードが中断した
2. NVIDIAのGPUドライバなど、DKMS（Dynamic Kernel Module Support）でビルドされる外部カーネルモジュールが、新カーネルに対するビルドに失敗していた（コンパイラの `-Werror` によって警告がエラー扱いとなり、ビルドが止まっていた）
3. `dpkg` 上ではこのカーネルパッケージが `iF`（*installed but failed to configure*）状態のまま残っていた
4. GRUBはデフォルトで最新（＝壊れている）カーネルを指しており、かつメニューのタイムアウトが `0` のため、リモートからは古いカーネルへの切り替えができなかった

つまり「新カーネルのinitramfsが存在しない」→「GRUBはそれでも新カーネルを起動しようとする」→「rootfsをマウントできずpanic」という流れである。

## 復旧手順

### 1. まずGRUBメニューからの回避を試す

ローカルコンソール（モニタ・キーボード）にアクセスできる場合は、まずGRUBメニューを表示できないか試す。ヘッドレス機で `GRUB_TIMEOUT=0` になっていると通常の起動ではメニューが一瞬も表示されないが、起動直後に `Shift` キーを押し続ける、あるいは `Shift+Insert` 同時押しでメニュー強制表示ができる場合がある。

メニューが表示できれば、Advanced options から**アップグレード前の旧カーネル**を選んで起動する。旧カーネルのinitramfsは既に存在しているはずなので、これだけで正常起動できる可能性が高い。起動できたら後述の「予防策」を適用して終了で良い。

今回のケースではこの方法でもメニューが出ず、Live USBでの復旧が必要になった。

### 2. Live USBでの起動

対象機のCPUアーキテクチャに合ったLive USBを用意する（x86_64機ならUbuntu Desktop/Serverのx86_64版、ARM64機ならarm64版、といった具合）。Ubuntu ServerのインストーラISOは「試用」モードがなくインストール専用の作りになっているため、**うっかりストレージ設定画面まで進めて確定してしまうと対象ディスクのデータが消える**点に注意。ディスク選択画面まで来たら、確定せずに `Ctrl+Alt+F2`（または `F3`）で別の仮想端末に切り替えると、インストール処理とは独立したシェルが得られる。

起動したら、まず対象ディスクの構成を確認する。

```bash
lsblk
sudo fdisk -l
```

USBメディア自体もブロックデバイスとして見えるので、内蔵ディスク（例: `nvme0n1` や `sda` の実ディスク側）とEFIパーティション・rootパーティションを取り違えないよう注意する。念のため一度マウントして中身を確認するとよい。

```bash
sudo mkdir -p /mnt
sudo mount /dev/<root-partition> /mnt
ls /mnt   # etc, boot, home, var などが見えればrootパーティションで確定
```

### 3. chroot環境の構築

```bash
sudo mount /dev/<esp-partition> /mnt/boot/efi
sudo mount --bind /dev  /mnt/dev
sudo mount --bind /dev/pts /mnt/dev/pts
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys  /mnt/sys
sudo chroot /mnt /bin/bash
```

### 4. 原因の特定

chroot内で `/boot` の中身とパッケージ状態を確認する。

```bash
ls /boot
dpkg -l | grep -E '^.[^i]'   # ii 以外(iF, iU など)の異常状態を抽出
```

`vmlinuz-*` はあるのに対応する `initrd.img-*` が無いバージョンが、壊れている新カーネルである。該当バージョンについて `dkms status` やビルドログ（`/var/lib/dkms/<module>/kernel-<version>-<arch>/log/make.log`）を確認すると、具体的な失敗原因がわかる。

ビルドログが巨大で読みにくい場合は `grep -in error` や `tail -n 60` で絞り込むとよい。ただし、サードパーティモジュールのソースコード自体が新しいカーネル/コンパイラに未対応であるようなケースでは、不便なLive USB linuxでビルドエラーを解消するのは現実的でない。

### 5. 壊れたカーネルを退避させ、旧カーネルにフォールバックする

今回は動作する（はずの）カーネルが存在しているため、無理にビルドを通そうとせず、**壊れた新カーネルのパッケージを削除し、initramfsが存在する旧カーネルにGRUBの選択肢を戻す**方が早い。

ネットワークが使えない環境（Live USBがオフライン)では `apt` はリポジトリ参照が絡んで失敗しやすいため、ローカル完結する `dpkg` を直接使う。

```bash
dpkg -l | grep <壊れたカーネルのバージョン>   # 正確なパッケージ名を確認
dpkg --purge linux-image-<broken-version>
dpkg --purge linux-modules-<broken-version>
dpkg --purge linux-modules-extra-<broken-version>
dpkg --purge linux-headers-<broken-version>
```

依存関係でエラーになる場合は、エラーメッセージに従って順序を入れ替えながら削除する。全て削除できたら設定を整合させる。上のパッケージ以外にも、壊れたバージョンに関連するlinux-tools- などがあれば削除しておく。

```bash
dpkg --configure -a
```

### 6. GRUB設定の再生成と再発防止

```bash
update-grub
```

出力に壊れていたバージョンが出てこず、正常なバージョンの `vmlinuz` / `initrd` が "Found" と表示されればOK。

合わせて、今後同様の事態が起きてもGRUBメニューから復旧できるよう、タイムアウト設定を有効化しておく。

```bash
sed -i 's/^GRUB_TIMEOUT_STYLE=.*/GRUB_TIMEOUT_STYLE=menu/' /etc/default/grub
sed -i 's/^GRUB_TIMEOUT=.*/GRUB_TIMEOUT=5/' /etc/default/grub
grep -q GRUB_TIMEOUT_STYLE /etc/default/grub || echo 'GRUB_TIMEOUT_STYLE=menu' >> /etc/default/grub
grep -q '^GRUB_TIMEOUT=' /etc/default/grub || echo 'GRUB_TIMEOUT=5' >> /etc/default/grub
update-grub
```

### 7. アンマウントして再起動

chrootを抜け、マウントした順序の逆順でアンマウントする。`umount -a` はLive環境自身のマウントまで巻き込みかねないため使わない。

```bash
exit  # chrootを抜ける
sudo umount /mnt/dev/pts
sudo umount /mnt/dev
sudo umount /mnt/proc
sudo umount /mnt/sys
sudo umount /mnt/boot/efi
sudo umount /mnt
mount | grep /mnt   # 何も出なければ全解除済み
sudo reboot
```

USBメディアを抜いて再起動し、正常にログイン画面まで到達すれば復旧完了。

## 予防策

- **ヘッドレス機でも `GRUB_TIMEOUT_STYLE=menu` を明示的に設定しておく。** `GRUB_TIMEOUT=0` はディスク容量的なメリットがほぼない割に、いざという時にリモートで詰む原因になる。
- **カーネルアップグレード直後にすぐ再起動しない。** `apt upgrade` 後は再起動前に `dkms status` で外部モジュールのビルドが全て成功しているか、`/boot` に新カーネルに対応する `initrd.img` が存在するかを確認する習慣をつける。
- サードパーティのカーネルモジュール（GPUドライバ等）に依存する環境では、`apt-mark hold` でカーネルパッケージの自動更新を止め、計画的にアップグレードする運用も検討する。
- 複数世代のカーネルを残しておく設定（`APT::Never-MarkAutoRemove` や `/etc/apt/apt.conf.d/` でのカーネル保持数の調整）にしておくと、フォールバック先が確保しやすい。

## まとめ

PCの「困ったら再起動」は多くの場合正しいが、カーネルアップデートなどが絡む場合気軽に再起動すると二度と起動しなくなるかもしれないのできちんと診断・巻き戻ししてから再起動する。
`unable to mount root fs` 系のkernel panicは、多くの場合「新カーネルのinitramfsが無い」「起動しようとしているカーネルが壊れている」ことが原因であり、Live USBからchrootして壊れたカーネルを退避させれば大抵は復旧できる。ただし、GRUBメニューが出ない設定になっていると復旧の難易度が跳ね上がるため、GRUBメニューは出るようにしておく方が安心である。GRUBメニューが出る状態であれば、古いカーネルから起動して新しい壊れたカーネルを消すだけで済んだかもしれない。
