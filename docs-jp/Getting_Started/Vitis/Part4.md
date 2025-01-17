<table class="sphinxhide">
 <tr>
   <td align="center"><img src="https://japan.xilinx.com/content/dam/xilinx/imgs/press/media-kits/corporate/xilinx-logo.png" width="30%"/><h1>2020.2 Vitis™ 入門チュートリアル</h1><a href="https://github.com/Xilinx/Vitis-Tutorials/tree/2020.1">2020.1 チュートリアルを参照</a></td>
 </tr>
 </table>

# Vitis フロー 101 – パート 4: 例のビルドおよび実行

このチュートリアルの 4 番目のパートでは、Vitis フローでサポートされる 3 つのビルド ターゲット (ソフトウェア エミュレーション、ハードウェア エミュレーション、およびハードウェア) を使用して、vector-add の例をコンパイルして実行します。

* ソフトウェア エミュレーション - カーネル コードは、ホスト プロセッサ上で実行されるようにコンパイルされます。これにより、ビルドと実行を高速に繰り返して反復アルゴリズムで調整していくことができます。このターゲットは、構文エラーを特定し、アプリケーションと共に実行されるカーネル コードをソース レベルでデバッグし、システムの動作を検証するのに便利です。

* ハードウェア エミュレーション - カーネル コードがハードウェア モデル (RTL) にコンパイルされ、専用シミュレータで実行されます。ビルドおよび実行ループにかかる時間は長くなりますが、詳細でサイクル精度のカーネル アクティビティが表示されます。このターゲットは、FPGA に配置するロジックの機能をテストして、初期パフォーマンス見積もりを取得する場合に便利です。

* ハードウェア - カーネル コードがハードウェア モデル (RTL) にコンパイルされ、FPGA にインプリメントされて、実際の FPGA で実行されるバイナリが生成されます。

データセンターとエンベデッド プラットフォームをターゲットにする場合、若干の違いがあります。ZCU102 カードと Alveo U200 カードの両方の手順は、次のとおりです。これらの手順は、ほかのカードでも簡単に使用できます。

> **重要**: このチュートリアルを実行するには、Vitis 2020.2 以降が必要です。

<details>
<summary><b>ZCU102 プラットフォームの手順については、ここをクリック</b></summary>

## エンベデッド プラットフォーム (ZCU102) でのビルドと実行

### 環境の設定

>**注記**: 次の手順は、bash シェルで実行していることを前提としています。

* Vitis を実行する環境を設定するには、次のスクリプトを実行します。

```bash
source <VITIS_install_path>/settings64.sh
source <XRT_install_path>/setup.sh
unset LD_LIBRARY_PATH
source $XILINX_VITIS/data/emulation/qemu/unified_qemu_v5_0/environment-setup-aarch64-xilinx-linux
```

* 次の環境変数が、それぞれ ZCU102 プラットフォーム、rootfs、および sysroot ディレクトリを指定するように設定されていることを確認します。

```bash
export PLATFORM_REPO_PATHS=<path to the ZCU102 platform install dir>
export ROOTFS=<path to the ZYNQMP common image directory, containing rootfs>
export SYSROOT=$ROOTFS/sysroots/aarch64-xilinx-linux
```

>**注記**: ZYNQMP 共通イメージ ファイルは、[Vitis エンベデッド プラットフォーム](https://japan.xilinx.com/support/download/index.html/content/xilinx/ja/downloadNav/embedded-platforms.html) ページからダウンロードできます。このファイルには、ザイリンクス Zynq MPSoC デバイスの sysroot、rootfs、およびブート イメージが含まれます。

### ソフトウェア エミュレーションをターゲットに指定

* ソフトウェア エミュレーション用にビルドするには、次のコマンドを入力します。

```bash
cd <Path to the cloned repo>/Getting_Started/Vitis/example/zcu102/sw_emu

aarch64-linux-gnu-g++ -Wall -g -std=c++11 ../../src/host.cpp -o app.exe -I${SYSROOT}/usr/include/xrt -L${SYSROOT}/usr/lib -lOpenCL -lpthread -lrt -lstdc++ --sysroot=${SYSROOT}
v++ -c -t sw_emu --config ../../src/zcu102.cfg -k vadd -I../../src ../../src/vadd.cpp -o vadd.xo
v++ -l -t sw_emu --config ../../src/zcu102.cfg ./vadd.xo -o vadd.xclbin
v++ -p -t sw_emu --config ../../src/zcu102.cfg ./vadd.xclbin --package.out_dir package --package.rootfs ${ROOTFS}/rootfs.ext4 --package.sd_file ${ROOTFS}/Image --package.sd_file xrt.ini --package.sd_file app.exe --package.sd_file vadd.xclbin --package.sd_file run_app.sh
```

次に、これら 5 つのコマンドを簡単に説明します。

1. `aarch64-linux-gnu-g++` は、Arm クロス コンパイラを使用してホスト アプリケーションをコンパイルします。
2. `v++ -c` は、vector-add アクセラレータのソース コードをコンパイルされたカーネル オブジェクト (.xo ファイル) にコンパイルします。
3. `v++ -l` は、コンパイルされたカーネルをターゲット プラットフォームにリンクし、FPGA バイナリ (.xclbin ファイル) を生成します。
4. `v++ -p` は、ホスト実行ファイル、rootfs、FPGA バイナリ、およびその他のいくつかのファイルをパッケージ化し、ブータブル イメージを生成します。

v++ ツールの -t オプションは、ビルド ターゲットを指定します。ここでは、ソフトウェア エミュレーション用にビルドしているため、 sw\_emu に設定されています。

また、--config オプションを使用すると、追加オプションを含むコンフィギュレーションファイルの名前を指定できます。ここでは、このコンフィギュレーション ファイルを使用して、ターゲット プラットフォームの名前を指定します。

```
platform=xilinx_zcu102_base_202020_1
save-temps=1
debug=1

# Enable profiling of data ports
[profile]
data=all:all:all
```

* ソフトウェア エミュレーションのビルドは迅速で、1 ～ 2 分以上かかることはありません。ビルド プロセスが完了したら、パッケージ ステップ (v++ -p) で生成された起動スクリプトを使用して、ソフトウェア エミュレーションを起動できます。

```bash
./package/launch_sw_emu.sh
```

* このコマンドは、ソフトウェア エミュレーションを起動し、Xilinx Quick Emulation (QEMU) を起動して、ブート シーケンスを開始します。Linux のブートが完了したら、次のコマンドを入力してサンプル プログラムを実行します。

```bash
mount /dev/mmcblk0p1 /mnt
cd /mnt
cp platform_desc.txt /etc/xocl.txt
export XILINX_XRT=/usr
export XILINX_VITIS=/mnt
export XCL_EMULATION_MODE=sw_emu
./app.exe
```

* 実行が正常に完了したことを示す次のメッセージが表示されます。

```bash
INFO: Found Xilinx Platform
INFO: Loading 'vadd.xclbin'
TEST PASSED
```

* Ctrl +A + X を押して QEMU を終了し、bash シェルに戻ります。

### ハードウェア エミュレーションをターゲットに指定

* ハードウェア エミュレーション用にビルドするには、次のコマンドを入力します。

```bash
cd ../hw_emu

aarch64-linux-gnu-g++ -Wall -g -std=c++11 ../../src/host.cpp -o app.exe -I${SYSROOT}/usr/include/xrt -L${SYSROOT}/usr/lib -lOpenCL -lpthread -lrt -lstdc++ --sysroot=${SYSROOT}
v++ -c -t hw_emu --config ../../src/zcu102.cfg -k vadd -I../../src ../../src/vadd.cpp -o vadd.xo
v++ -l -t hw_emu --config ../../src/zcu102.cfg ./vadd.xo -o vadd.xclbin
v++ -p -t hw_emu --config ../../src/zcu102.cfg ./vadd.xclbin --package.out_dir package --package.rootfs ${ROOTFS}/rootfs.ext4 --package.sd_file ${ROOTFS}/Image --package.sd_file xrt.ini --package.sd_file app.exe --package.sd_file vadd.xclbin --package.sd_file run_app.sh
```

* 前の手順との唯一の違いは、v++ ターゲット (-t) オプションで、sw\_emu から hw\_emu に変更されます。その他のオプションはすべて同じままです。

* ハードウェア エミュレーションのビルドには約 5 分かかります。ビルド プロセスが完了したら、パッケージ ステップで生成された起動スクリプトを使用して、ハードウェア エミュレーションを起動できます。

```bash
./package/launch_hw_emu.sh
```

* Linux の起動が完了したら、QEMU コマンド ラインに次のコマンドを入力してサンプル プログラムを実行します。

```bash
mount /dev/mmcblk0p1 /mnt
cd /mnt
cp platform_desc.txt /etc/xocl.txt
export XILINX_XRT=/usr
export XILINX_VITIS=/mnt
export XCL_EMULATION_MODE=hw_emu
./app.exe
```

* TEST PASSED というメッセージが表示され、実行が正常に完了したことが示されます。

* Ctrl +A + X を押して QEMU を終了し、bash シェルに戻ります。

### ハードウェア ターゲットの指定

* ハードウェア用にビルドするには、次のコマンドを入力します。

```bash
cd ../hw

aarch64-linux-gnu-g++ -Wall -g -std=c++11 ../../src/host.cpp -o app.exe -I${SYSROOT}/usr/include/xrt -L${SYSROOT}/usr/lib -lOpenCL -lpthread -lrt -lstdc++ --sysroot=${SYSROOT}
v++ -c -t hw --config ../../src/zcu102.cfg -k vadd -I../../src ../../src/vadd.cpp -o vadd.xo
v++ -l -t hw --config ../../src/zcu102.cfg ./vadd.xo -o vadd.xclbin
v++ -p -t hw --config ../../src/zcu102.cfg ./vadd.xclbin --package.out_dir package --package.rootfs ${ROOTFS}/rootfs.ext4 --package.sd_file ${ROOTFS}/Image --package.sd_file xrt.ini --package.sd_file app.exe --package.sd_file vadd.xclbin --package.sd_file run_app.sh
```

* ハードウェアをターゲットにするには、v++ -t オプションを hw に設定します。その他のオプションはすべて同じままです。
* ハードウェアのビルドには約 30 分かかりますが、正確な所要時間は、ビルドするマシンとその負荷によって異なります。
* ビルド プロセスが完了したら、sd\_card ディレクトリを SD カードにコピーし、プラットフォームに接続して、Linux プロンプトが表示されるまで起動します。この時点で、次のコマンドを入力してアクセラレーションされたアプリケーションを実行します。

```bash
mount /dev/mmcblk0p1 /mnt
cd /mnt
cp platform_desc.txt /etc/xocl.txt
export XILINX_XRT=/usr
export XILINX_VITIS=/mnt
./app.exe
```

* 実行が正常に完了したことを示す TEST PASSED という同じメッセージが表示されます。
* これで、ZCU102 カードで初めて Vitis アクセラレーション アプリケーションを実行できました。

</details>
<details>
<summary><b>Alveo U200 プラットフォームの手順については、ここをクリック</b></summary>

## データセンター プラットフォーム (U200) でのビルドおよび実行

### 環境の設定

>**注記**: 次の手順は、bash シェルで実行していることを前提としています。

* Vitis を実行する環境を設定するには、次のスクリプトを実行します。

```bash
source <VITIS_install_path>/settings64.sh
source <XRT_install_path>/setup.sh
```

* 次の環境変数が、U200 プラットフォーム、インストール ディレクトリを指定するように設定されていることを確認します。

```bash
export PLATFORM_REPO_PATHS=<path to the U200 platform install dir>
```

### ソフトウェア エミュレーションをターゲットに指定

* ソフトウェア エミュレーション用にビルドするには、次のコマンドを入力します。

```bash
cd <Path to the cloned repo>/Getting_Started/Vitis/example/u200/sw_emu

g++ -Wall -g -std=c++11 ../../src/host.cpp -o app.exe -I${XILINX_XRT}/include/ -L${XILINX_XRT}/lib/ -lOpenCL -lpthread -lrt -lstdc++
emconfigutil --platform xilinx_u200_xdma_201830_2 --nd 1
v++ -c -t sw_emu --config ../../src/u200.cfg -k vadd -I../../src ../../src/vadd.cpp -o vadd.xo
v++ -l -t sw_emu --config ../../src/u200.cfg ./vadd.xo -o vadd.xclbin
```

次に、これら 4 つのコマンドを簡単に説明します。

1. `g++` は、標準 GNU C コンパイラを使用してホスト アプリケーションをコンパイルします。
2. `emconfigutil` は、指定したプラットフォーム用にエミュレーションするデバイスのタイプと数を定義するエミュレーション コンフィギュレーション ファイルを生成します。
3. `v++ -c` は、vector-add アクセラレータのソース コードをコンパイルされたカーネル オブジェクト (.xo ファイル) にコンパイルします。
4. `v++ -l` は、コンパイルされたカーネルをターゲット プラットフォームにリンクし、FPGA バイナリ (.xclbin ファイル) を生成します。

v++ ツールの -t オプションは、ビルド ターゲットを指定します。ここでは、ソフトウェア エミュレーション用にビルドしているため、 sw\_emu に設定されています。

また、--config オプションを使用すると、追加オプションを含むコンフィギュレーションファイルの名前を指定できます。ここでは、このコンフィギュレーション ファイルを使用して、ターゲット プラットフォームの名前と DDR バンクのカーネル引数のマッピングを指定します。

```
platform=xilinx_u200_xdma_201830_2
debug=1
save-temps=1

[connectivity]
nk=vadd:1:vadd_1
sp=vadd_1.in1:DDR[1]
sp=vadd_1.in2:DDR[2]
sp=vadd_1.out:DDR[1]

[profile]
data=all:all:all
```

* ソフトウェア エミュレーションのビルドは迅速で、1 ～ 2 分以上かかることはありません。ビルド プロセスが完了したら、次のようにソフトウェア エミュレーションを開始できます。

```bash
export XCL_EMULATION_MODE=sw_emu
./app.exe
```

* 実行が正常に完了したことを示す次のメッセージが表示されます。

```bash
INFO: Found Xilinx Platform
INFO: Loading 'vadd.xclbin'
TEST PASSED
```

### ハードウェア エミュレーションをターゲットに指定

* ハードウェア エミュレーション用にビルドするには、次のコマンドを入力します。

```bash
cd ../hw_emu

g++ -Wall -g -std=c++11 ../../src/host.cpp -o app.exe -I${XILINX_XRT}/include/ -L${XILINX_XRT}/lib/ -lOpenCL -lpthread -lrt -lstdc++
emconfigutil --platform xilinx_u200_xdma_201830_2 --nd 1
v++ -c -t hw_emu --config ../../src/u200.cfg -k vadd -I../../src ../../src/vadd.cpp -o vadd.xo
v++ -l -t hw_emu --config ../../src/u200.cfg ./vadd.xo -o vadd.xclbin
```

* 前の手順との唯一の違いは、v++ ターゲット (-t) オプションで、sw\_emu から hw\_emu に変更されます。その他のオプションはすべて同じままです。

* ハードウェア エミュレーションのビルドには約 5 ～ 6 分かかります。ビルド プロセスが完了したら、次のようにハードウェア エミュレーションを起動できます。

```bash
export XCL_EMULATION_MODE=hw_emu
./app.exe
```

* 実行が完了すると、問題なく実行が終了したことを示す TEST PASSED というメッセージが表示されます。

### ハードウェア ターゲットの指定

* ハードウェア用にビルドするには、次のコマンドを入力します。

```bash
cd ../hw

g++ -Wall -g -std=c++11 ../../src/host.cpp -o app.exe -I${XILINX_XRT}/include/ -L${XILINX_XRT}/lib/ -lOpenCL -lpthread -lrt -lstdc++
v++ -c -t hw --config ../../src/u200.cfg -k vadd -I../../src ../../src/vadd.cpp -o vadd.xo
v++ -l -t hw --config ../../src/u200.cfg ./vadd.xo -o vadd.xclbin
```

* ハードウェアをターゲットにするには、v++ -t オプションを hw に設定し、emconfigutil ステップはエミュレーションにのみ適用されるためスキップされます。その他のオプションはすべて同じままです。
* ハードウェアのビルドには約 1.5 ～ 2 時間かかることがありますが、正確な所要時間は、ビルドするマシンとその負荷によって異なります。
* ビルドが完了したら、U200 カードを使用してアプリケーションを高速化できます。

```bash
./app.exe
```

>**注記**: Alveo カードがインストールされたサーバーでプログラムを実行してください。アプリケーションを別のマシン上にビルドした場合は、目的のサーバーに接続して上記のコマンドを実行する前に、/opt/xilinx/xrt/setup.sh スクリプトを読み込む必要があります。

* 実行が正常に完了したことを示す TEST PASSED という同じメッセージが表示されます。
* これで、Alveo U200 カードで初めて Vitis アクセラレーション アプリケーションを実行できました。

</details>

## 次の手順

はじめての例を実行したので、このチュートリアルの[**パート 5**](./Part5.md) に進んで、Vitis アナライザーでアプリケーションを視覚化してプロファイルを作成する方法を学びます。

<p align="center"><sup>Copyright&copy; 2020 Xilinx</sup></p>
