# RDMAを用いた，遠隔ベアメタルマシンの仮想アドレス空間の復元

## 背景

- OSデバッグやセキュリティフォレンジックするために物理・仮想メモリの解析が利用される
  - 通常OSのメモリ解析には仮想マシン(VM)を利用
  - XenやQEMU＋KVMなど
    - 関連ソフトウェア）libvmi, google/rekallなど

## 課題

- 物理マシンに対して，カーネルパニック時におけるメモリ解析手段が存在しない
  - 内部リソース・ロジックを使用するため

## 目的

- 遠隔ベアメタルマシンの物理メモリの監視アーキテクチャを確立
  - 外部リソースで動作

## 実験環境

- 監視プロセスが動くホスト
  - x86-64 Linux
- 監視対象ホスト
  - x86-64 Linux

## アプローチ

- カーネルパニック時に特定のプロセスの仮想アドレス空間を復元する

### アーキテクチャの説明

#### 環境の説明

図をここに挿入

#### PCIe Eth Bridgeの説明

- 物理アドレスを指定することで，値を4Byteずつ取ることができるFPGAデバイス(soraさん実装)
- 本研究においては，このデバイスの手続きを大量に呼び出すことにより，連続した領域の値を取得

## x86_64 LinuxのMMUの仕組み

### 論理・リニア・物理アドレスの関係

- セグメント機構
  - 論理アドレスからリニアアドレスを算出する機構
- ページング機構
  - リニアアドレスから物理アドレスを算出する機構

### Linuxアーキテクチャの説明

- セグメント機構と，ページング機構がプロセッサに用意されている
  - Linuxではページング機構のみ使用
- プロセッサの制約上，セグメント機構は無効にできない
  - フラットになるように設定されている

- Linuxのプロセスでは，アドレスの参照にリニアアドレスを使用
- 実際のメモリの読み書きは，ページング機構を通し，物理アドレスを算出している

### 各階層の説明

- Page Map Level 4
  - Page Directory Pointer
    - Page Directory
      - Page Table

- テーブルである
- 64bit Linuxにおいては，512個のエントリがそれぞれある．
- 1エントリのサイズは8 Byte
- テーブル全体のサイズは，`8 Byte * 512 = 4KB`

#### テーブルとエントリの値

テーブルとエントリの図をここに置く

- Pはエントリの再帰に物理アドレスか設定されているかどうかのフラグ．
  - 空のエントリの場合，ここが0となる
- Page Table のエントリには物理アドレスが格納
  - 4KB境界

### CR3を交えた部分

- プロセスごとに，4階層のテーブルをもち，アドレス空間を独立させている
  - **CR3レジスタ**に，ページングの起点となるPage Map Level 4のベースアドレスが保持されている．

CR3レジスタの構造を示す図

### プロセスが物理アドレスを算出する流れ

仮想アドレスのどのビットがどのインデックスに相当するかを書いた図を貼る

- 仮想アドレスが0x7fffffffe358の場合，インデックスとオフセットは以下の通り
  - PML4 → 255
  - Pointer → 511
  - Directory → 511
  - Table → 510
  - Offset → 856

- 前述のテーブルの各エントリにアクセス
  - Tableの39-12bitの値(4KB境界)にOffsetの値を足す

## 実装

### 逆算する

本研究においては，以上の手順を逆算，4階層全てのテーブルの情報を本研究の実装で保持し，仮想アドレス空間を復元する

- CR3はレジスタであるため，現在の値をメモリから参照することができない
- CR3の値を監視対象ホストから通知
- 通知された値を起点に，4階層全てのデータを読み込み
  - 愚直にやると，512 ** 4個のエントリを読み込む必要が出てくる．
  - エントリごとに，Pフラグが0の場合は次の階層からスキップ

### 具体的な実装の流れ

- PML4, Pointer, Directory, Tableの各エントリから，次の階層のアドレスを取得
- Tableまで到達したら，物理アドレスの保持．(Page Tableの値まではポインタ)

- 物理アドレスを直接指定し，アドレスを保持．(物理アドレスの値が実際の値)
- 以下の構造体に格納

```c
typedef struct _BINARY
{
    uint64_t pml4Index;
    uint64_t pdpIndex;
    uint64_t pdIndex;
    uint64_t ptIndex;
    uint64_t offset;
    uint64_t addr;
    uint64_t virtAddr;
    uint64_t value;
} BINARY;
```

ここにある値(`value`)を最後に復元する

```c
(pml4Index << 39)+ (pdpIndex << 30) + (pdIndex << 21) + (ptIndex << 12) + offset;
```

仮想アドレスの復元が完了

## 評価

- カーネルパニック時の対象プロセスの仮想アドレスの復元
- Kernel Panicがおきた時間の特定

### 監視するプロセスの説明

[getcr3/user.c](https://github.com/dooooooooinggggg/getcr3/blob/master/user.c) (macchanさん実装のカーネルモジュールを拡張したものをinclude)

- CR3の値をカーネルモジュール経由でprintf()
- 現在のUNIX TIMEをprintf()

user.c

```c
while (1)
    {
        time_t a = time(NULL); // UNIX TIME
        uint64_t aVirtAddr = &a;

        printf("a:%ld(0x%lx) at %p\n", a, a, &a);
        printIndex(aVirtAddr);
        rc = read(fd, cr, 40);
        for (int i = 0; i < CR_NUMBER; i++)
        {
            // cr1 cannot see
            if (i == 1)
                continue;
            printf("CR%d: 0x%" PRIx64 "\n", i, cr[i]);
        }
        sleep(1);
    }
}
```

### 1.azkaban(監視対象ホスト)

```sh
$ ./user
# .
# .
# .
# .
# a:1548316892(0x5c4970dc) at 0x7fff6509aa90
# 255 509 296 154 2704
# CR0: 0x80050033
# CR2: 0x7fbf309c3230
# CR3: 0x274be2000
# CR4: 0x160670

# a:1548316893(0x5c4970dd) at 0x7fff6509aa90
# 255 509 296 154 2704
# CR0: 0x80050033
# CR2: 0x7fbf309c3230
# CR3: 0x274be2000
# CR4: 0x160670

# a:1548316894(0x5c4970de) at 0x7fff6509aa90     <- 最後の出力
# 255 509 296 154 2704
# CR0: 0x80050033
# CR2: 0x7fbf309c3230
# CR3: 0x274be2000
# CR4: 0x160670
```

### 2. azkaban(監視対象ホスト) 別コンソール

```sh
$ sudo su -
root$ echo c > /proc/sysrq-trigger
```

->azkaban(監視対象ホスト)がカーネルパニックで停止．

### 3. リモートホストからメモリの読み込みを開始．

```sh
tatsu@quantumcomputer:~/dmaLinux$ sudo ./dmaLinux 0x274be2000
# ==================================
# This is dmaLinux
# Remote Direct Memory Access to Linux and
# Debug virt address and process
# x86_64 architecture only
# ==================================

# pml4BaseAddress :0x274be2000
# prease enter i1start,i1end
# 254,256                                                <- 出力結果を元に自分で入力
# pml4[254]:0x274be27f0: 0x0
# pml4[255]:0x274be27f8: 0x8000000273954067
# pml4[256]:0x274be2800: 0x0
# PML4 Done
# prease enter i2start,i2end
# 508,510                                                <- 出力結果を元に自分で入力
# pdpe[255][508]:0x273954fe0: 0x0
# pdpe[255][509]:0x273954fe8: 0x273910067
# pdpe[255][510]:0x273954ff0: 0x0
# Page directory pointer Done
# prease enter i3start,i3end
# 295,297                                                <- 出力結果を元に自分で入力
# pde[255][509][295]:0x273910938: 0x0
# pde[255][509][296]:0x273910940: 0x274b52067
# pde[255][509][297]:0x273910948: 0x0
# Page directory Done
# prease enter i4start,i4end
# 152,156                                                <- 出力結果を元に自分で入力
# pte[255][509][296][152]:0x274b524c0: 0x0
# pte[255][509][296][153]:0x274b524c8: 0x80000002697c9867
# pte[255][509][296][154]:0x274b524d0: 0x80000002697e5867
# pte[255][509][296][155]:0x274b524d8: 0x8000000269cba865
# pte[255][509][296][156]:0x274b524e0: 0x0
# Page table Done
# Phys Addr Read from Page Table Entry
# prease offset_start,offset_range
# 0,40                                                <- 出力結果を元に自分で入力
# virt:0x7fff65099000, phys:0x2697c9000: 0x0
# virt:0x7fff65099008, phys:0x2697c9008: 0x0
# virt:0x7fff65099010, phys:0x2697c9010: 0x0
# virt:0x7fff65099018, phys:0x2697c9018: 0x0
# virt:0x7fff65099020, phys:0x2697c9020: 0x0
# prease offset_start,offset_range
# 2700,2740                                                <- 出力結果を元に自分で入力
# virt:0x7fff6509aa8c, phys:0x2697e5a8c: 0x5c4970de00000028
# virt:0x7fff6509aa94, phys:0x2697e5a94: 0x6509aa9000000000
# virt:0x7fff6509aa9c, phys:0x2697e5a9c: 0x8005003300007fff
# virt:0x7fff6509aaa4, phys:0x2697e5aa4: 0x75cf3b0000000000
# virt:0x7fff6509aaac, phys:0x2697e5aac: 0x309c3230ffff8802
# prease offset_start,offset_range
# 0,40                                                 <- 出力結果を元に自分で入力
# virt:0x7fff6509b000, phys:0x269cba000: 0x0
# virt:0x7fff6509b008, phys:0x269cba008: 0x0
# virt:0x7fff6509b010, phys:0x269cba010: 0x0
# virt:0x7fff6509b018, phys:0x269cba018: 0x0
# virt:0x7fff6509b020, phys:0x269cba020: 0x0
# dmaRead done
# -------------RESULT-------------
# 0x7fff65099000(phys[0x2697c9000]) : 0x0
# 0x7fff65099008(phys[0x2697c9008]) : 0x0
# 0x7fff65099010(phys[0x2697c9010]) : 0x0
# 0x7fff65099018(phys[0x2697c9018]) : 0x0
# 0x7fff65099020(phys[0x2697c9020]) : 0x0
# 0x7fff6509aa8c(phys[0x2697e5a8c]) : 0x5c4970de00000028 <- 停止した時間が格納されている領域
# 0x7fff6509aa94(phys[0x2697e5a94]) : 0x6509aa9000000000
# 0x7fff6509aa9c(phys[0x2697e5a9c]) : 0x8005003300007fff
# 0x7fff6509aaa4(phys[0x2697e5aa4]) : 0x75cf3b0000000000
# 0x7fff6509aaac(phys[0x2697e5aac]) : 0x309c3230ffff8802
# 0x7fff6509b000(phys[0x269cba000]) : 0x0
# 0x7fff6509b008(phys[0x269cba008]) : 0x0
# 0x7fff6509b010(phys[0x269cba010]) : 0x0
# 0x7fff6509b018(phys[0x269cba018]) : 0x0
# 0x7fff6509b020(phys[0x269cba020]) : 0x0
```

0x7fff6509aa8c(phys[0x2697e5a8c]) : 0x**5c4970de**00000028 <- 停止した時間が格納されている領域

### 4. 結論

- (CR3の値がわかっている状態であれば，)カーネルパニックが起きていても，仮想アドレス空間を復元できることを確認．
- user.cの最後の出力と一致している
  - リモートホストから，カーネルパニックが起きた時間を特定することができたということ

## 今後の展望

- CR3レジスタはtask_struct->mm_struct->pgdに格納されている
  - カーネル空間にあるこの変数を外から探す
  - 全プロセスの復元が可能になる
- デバイスを修正して全てのテーブルを検査できるように修正する．
  - indexやoffset自分で指定し入力しているのは，環境的な問題
