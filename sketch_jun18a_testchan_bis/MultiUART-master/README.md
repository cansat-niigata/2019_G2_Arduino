# MultiUART

復数のUARTデバイスを扱える SoftwareSerial の代替実装（for Arduino IDE）

## 概要

Arduino標準の SoftwareSerialは、とても良く出来てはいるものの以下の制約がある。

- 復数のUARTデバイスから同時に受信することが出来ない。
ただひとつの listen されたポートからのみ受信データを読み取ることができ、
他のポートからの着信はすべて無視され、バッファリングもされない。
- その使用の有無に関わらず、すべての ISR(PCINTn_vect) が専有される。
よって汎用ポート群レベル割込との併用はできない。
- 無通信時はなんら負荷とはならないが、
送信／受信処理中は 100%のCPUクロックを消費し、他の一切の割込処理が停止または留保される。

本Arduino向けライブラリは、これに対して以下の機能を提供する。

- 最大4つまでの（充分に低速な）UARTデバイスからの受信データを同時にバッファできる。
- ISR(PCINTn_vect) を使用しないため、汎用ポート群レベル割込が併用できる。
- SoftwareSerial とは互いに干渉しないため、CPUリソースが許す限り、同時使用が出来る。
よって従来のスケッチに第2第3の UARTを追加する用途にも応える。
- HardwareSerialへの透過的ブリッジとしても使用でき、
コンストラクタ初期化の修正だけでソフトウェア／ハードウェア双方の
UARTに対応したアプリケーションを書ける。

その一方で、本ライブラリは以下の制約を持つ。

- ボーレート生成のため、標準では TIMER1 を専有する。
これはマクロ指示子によって TIMER2 に代えることは可能。
- （TIMER1を使う限り）ATmega328P / 32U4 / 1284P 等主要な MCUで動作可能。
- 調歩同期のための受信割込処理は非常に重く、16Mhz駆動時でも40%以上の CPUタイムを常に消費する。
16Mhz未満の、あるいは内蔵RC発振器を使用している場合、多くは期待通りに動作しない。
この点で SoftwareSerial の割り切った実装は理に適っている。

## Arduino IDE への導入

1. .ZIPアーカイブをダウンロードする。[Click here](https://github.com/askn37/MultiUART/archive/master.zip)

2. ライブラリマネージャで読み込む

  スケッチ -> ライブラリをインクルード -> .ZIP形式のライブラリをインストール...

## 使い方

### とっかかり

SoftwareSerial と同様に、コンストラクタで入出力に使うピンを指定し、
Arduinoのセットアップで MultiUART::begin() にボーレートを指定して呼び出す。

```c
#include <MultiUART.h>

#define RX_PIN (8)
#define TX_PIN (9)

MultiUART UART(RX_PIN, TX_PIN);

void setup() {
  Serial.begin(9600);
  UART.begin(9600);
}

void loop() {
  if (Serial.available()) {
    UART.write(Serial.read());
  }
  while (UART.available()) {
    Serial.write(UART.read());
  }
}
```

### 復数UARTの並列使用

最大4つまでのUARTを同時に開くことが出来る。

```c
MultiUART GPS(GPS_RX_PIN, GPS_TX_PIN); GPS.begin(9600);
MultiUART BLE(BLE_RX_PIN, BLE_TX_PIN); BLE.begin(9600);
MultiUART CON(CON_RX_PIN, CON_TX_PIN); CON.begin(9600);
MultiUART AUX(AUX_RX_PIN, AUX_TX_PIN); AUX.begin(9600);
```

ただし4つのUARTが、必ずしも同時に、完全に、扱えるとは限らない。
MCUの処理能力（CPUクロックや割込頻度）によっては受信データの欠落が発生しうる。
合算帯域は 38400 を超えるべきではない。

ボーレートのより高い UARTデバイスには優先的に begin() を発行して利用開始することが望ましい。
ただし stopListening() と listen() を繰り返したのちの、優先保証はされない。

### 指定可能なボーレート

begin() に指定できるボーレートは、通常の使用範囲では 9600(bps)が最大である。
実用上の最小値は 768(bps)で、その間には 19200 の約数が指定できる。

    9600 6400 4800 3840 3200 2400 1920 1600 1280 1200 960 800 768

# リファレンス

ここでは Arduinoスタイルではなく、型を含めた C/C++スタイルで各メソッドを記述する。

## コンストラクタ

### MultiUART (uint8\_t RX\_PIN, uint8\_t TX\_PIN)

クラスオブジェクト生成にはこのコンストラクタを使用する。
受信ピン、送信ピンの指定は必須である。
ピン番号は Arudiono で規定されたものとする。
それぞれのピンのIO設定は直ちに行われる。

```c
// もっぱらグローバルなオブジェクトとして
MultiUART UART(RX_PIN, TX_PIN);
UART.begin(9600);

// スコープから抜けると破棄されるようなオブジェクトとして
auto UART = new MultiUART(RX_PIN, TX_PIN);
UART->begin(9600);
```

本クラスは Stream クラスを継承している。
よって Print クラスも使用できる。

SoftwareSerial と異なり、Invertedフラグは指定できない。
ビット解釈は HIGH=1、LOW=0 に固定である。

受信ピン指定は、ユニークでなければならない。
復数のオブジェクトで同じ受信ピン指定を行うと混乱した結果を招く。
一方で、送信ピン指定は復数のオブジェクトで共用しても（実用上はどうであれ）一向に構わない。

無効なピン番号（例えば -1）が与えられるとその機能は使用されない。
従って受信のみ、あるいは送信のみが有効なオブジェクトを作ることが出来る。

```c
MultiUART TxOnly(-1, TX_PIN);			// 送信専用で write() だけが機能する
MultiUART RxOnly(RX_PIN, -1);			// 受信専用で write() は何もしない
MultiUART RxOnly2(RX_PIN, RX_PIN);		// これも受信専用
```

受信ピンと送信ピンに同じピン番号を与えると、送信ピン指定は無視されて、受信専用に設定される。

### MultiUART (HardwereSerial& SERIAL)

通常のコンストラクタ初期化に代えて、HardwereSerialクラスオブジェクトを渡すと
MultiUARTは HardwereSerialへの透過的ブリッジとして機能するようになる。

```c
// もっぱらグローバルなオブジェクトとして
MultiUART CON(Serial1);
CON.begin(9600);

// スコープから抜けると破棄されるようなオブジェクトとして
auto CON = new MultiUART(Serial1);
CON->begin(9600);
```

この機能は MultiUARTクラスを継承した子孫クラスを作成・使用した場合、
その子孫クラスに MultiUARTによるソフトウェアシリアルと、
AVRによるハードウェア支援のされたハードウェアシリアルを
選択する自由を与える。

MultiUARTの機能のうち、last() や setRxBuffer() 等が使用可能になる。
一方で以下のメソッドは機能しないか意味をなさない。

    overflow() isFraming() listen() isListening()
    stopListening() stopListener() setThrottle()

以下のメソッドは MultiUART では機能しないが、HardwareSerialブリッジでは機能する

    flush() availableForWrite()

同時使用可能な UART数に、HardwareSerialブリッジは含まれない。

受信バッファは MultiUART と HardwareSerial 両方の合計となる。
（available() 等の読込系メソッドを実行することで、MultiUART のバッファに転送される）

本機能は 0.9.6 で実装された。

### operator bool (void)

真偽値として評価された場合、通信準備ができているかを返す。
実際には常に真を返す。

```c
if (UART) Serial.println("true");
```

ただし HardwareSerial ブリッジとして使用している場合は、
HardwareSerial の返す真偽値となる。

## リソース制御

### bool begin (long BAUDRATE, uint8_t config = SERIAL_8N1)

オブジェクトにボーレートを設定し、受信処理を開始する。
（HardwareSerialブリッジ時を除き）
すでに同時受信利用可能UART数を超えている場合は偽を返し、受信処理は有効にならない。

```c
if (!UART.begin(9600)) Serial.println("not ready");
```

これは listen() が失敗したことを示すので、
他のオブジェクトに対して stopListening() （あるいはデストラクタ）を発行してから
listen() を実行することでエラー状態から復せる。

第2引数はシリアルポートとしての設定であるが MultiUARTでは 8N1 に固定である。
HardwareSerialブリッジとした場合は省略することも、任意の指定もできる。

便宜上（好ましくはないことだが）第1引数まですべて省略してもエラーにはならない。
既定値として 9600 が採用される。

### bool listen (void)

このオブジェクトの受信処理を開始する。
すでに同時受信利用可能UART数を超えている場合は偽を返す。

begin() からは暗黙的に呼ばれる。

SoftwareSerial の同名メソッドとは異なり、他のオブジェクトを自動的に利用不可とはしない。
先だって使用停止したいオブジェクトに対し明示的に stopListening() を発行する必要がある。

HardwareSerial ブリッジの場合は意味をなさず、常に真を返す。

### bool isListening (void)

このオブジェクトが受信可能かを返す。listen() していなければ偽を返す。

HardwareSerial ブリッジの場合は意味をなさず、常に真を返す。

### bool stopListening (void)

このオブジェクトの受信処理を停止する。
停止に成功すると真を、既に停止している場合は偽を返す。
同時受信利用可能UART数はひとつ空く。

受信を停止していても送信には影響しない。

HardwareSerial ブリッジの場合は意味をなさず、常に偽を返す。

### void end (void);

オブジェクトの受信処理を停止する。
stopListening() と違い返値を戻さない。
同時受信利用可能UART数はひとつ空く。
使用していた IOピンは未使用になる。

受信を停止していても送信には影響しない。

デストラクタからは暗黙的に呼ばれる。

HardwareSerial ブリッジの場合は
HardwareSerial のデストラクタが呼ばれ、対応するIOピンが未使用になる。

## 受信機能

### int read (void)

使用している受信バッファから1キャラクタを読んで返す。
読んだ文字は受信バッファから除かれる。
バッファが空の場合は -1 が返される。

### int peek (void)

使用している受信バッファから先頭の1キャラクタを読んで返す。
読んだ文字は受信バッファから除かれずに残る。
バッファが空の場合は -1 が返される。

### int last (void)

使用している受信バッファから末尾の1キャラクタを読んで返す。
読んだ文字は受信バッファから除かれずに残る。
バッファが空の場合は -1 が返される。

このメソッドは MultiUART に特有のもので、
受信バッファからキャラクタを取り出す前に、
改行文字などのデリミタが受信バッファに投入されたかを
事前に調べる機会を与える。

こもメソッドは 0.9.3 で実装された。

### int available (void);

使用している受信バッファに貯められている文字数を返す。
バッファが空であれば 0を返す。

### bool overflow (void)

使用している受信バッファが溢れているなら真を返す。
溢れフラグはこのメソッドを呼ぶことでクリアされる。

名称が isOverflow() でないのは、SoftwareSerial と互換性を保つためである。

HardwareSerial ブリッジに対しては機能せず、常に偽を返す。

### bool isFraming (void)

フレーム受信途中であるなら真を返す。
すなわちスタートビットを認識し、ストップビットがまだ判定されていないことを示す。

HardwareSerial ブリッジに対しては機能せず、常に偽を返す。

## 送信機能

### size\_t write (const uint8\_t character)

指定したキャラクタ文字を送信する。
本ライブラリの実装では送信バッファを持たないため、送信は都度、同期的に行われる。
返値は常に 1である。

isListening() が偽を返しても、受信と送信は独立しているので、関係なく送信できる。

送信は基準クロックに従って実行される。
よって割込が禁止されているかあるいは TIMERが停止しているあいだ、本メソッドは何もしない。

また遅いボーレートに対してはより多くの時間を、割込対応以外の何も出来ない waitに割くこととなる。
現在のところ長いビット送信の合間に yield() を呼ぶような救済処置は実装されていない。

### void flush (void)

送信バッファがすべて書き出されるまで待つ。
しかしながら本ライブラリは送信バッファを持たないので、このメソッドは何もせずにすぐ戻る。

HardwareSerial ブリッジに対しては額面通りに機能する。

### int availableForWrite (void)

送信バッファの空き文字数を返す。
しかしながら本ライブラリは送信バッファを持たないので、このメソッドは常に 1を返す。

HardwareSerial ブリッジに対しては額面通りに機能する。

## ユーティリティ／実験的実装

### uint8_t getBaseClock (void)

基準クロックでインクリメントされている 8bitカウンタの値を返す。
分解能は既定値で 52us、13.3ms強で1周している。
これよりも長く低速なボーレートと、分解能の2倍に満たない高速なボーレートには対応できない。

### void stopListener (void)

基準クロック割込を停止し、すべての UARTも停止する。
これはオブジェクト個別に stopListening() を行うよりも強力に実施される。

MultiUART の受信処理は待機状態でも非常に重く、
16Mhz駆動でも 40%以上の CPUタイムを消費する。
このメソッドは MultiUART の機能をすべて止めて、CPUタイムを空けるためにある。

```c
// クラスメソッドとして
MultiUART::stopListener();
// またはオブジェクトメソッドとして
UART.stopListener();
```

受信処理を再開するには、各オブジェクトに対して個別に listen() を使用する。

### void setRxBuffer (volatile char* BUFFER, int LENGTH);

実験的実装：
オブジェクトに外部の受信バッファを設定する。
与える引数は char型のバッファ先頭アドレスと、バッファ長である。
バッファ長は2のべき乗数でかつ 256以下でなければならない。
従って指定できる値は実用上、以下の値に限られる。
これら以外の指定は意図した結果を得られないだろう。

    256 128 (64 32 16)

このメソッドを用いても、オブジェクトが持つ規定のバッファがメモリから開放されることはない。
その既定値は 64であるため、この機能をが必要になる場面はそう多くはない。

```c
static volatile char MyBuffer[256];

// 現在のバッファが空になるのを待つ
while (UART.available() || UART.isFraming()) UART.read();

// バッファを設定する
UART.setRxBuffer(MyBuffer, 256);

// バッファ設定を規定値に戻す（リセットする）
UART.setRxBuffer();
```

メソッドを空引数で呼ぶと、外部バッファを切り離し、オブジェクトが持つ規定の受信バッファに設定を戻す。

バッファの設定は、現在受信途中のデータがあれば喪失することを意味する。

使用中の外部バッファには、直接アクセスすべきではない。
available() と read() を用いて読み出すべきである。
しかしながら外部バッファが溢れる前に外部バッファを切り離し、
末尾に \\0 を補完すれば通常の文字列として扱うことは出来る。

```c
// バッファを設定する
UART.setRxBuffer(MyBuffer, 256);

// コマンドを送って応答を待つ
UART.println("hello!");
while (UART.last() == '\n');

// バッファを切断する前に通信を停止して、ステータスを得る
UART.stopListening();
int len = UART.available();
vool overflow = UART.overflow();

// バッファを切断して溢れていなければ、内容を文字列として解釈
UART.setRxBuffer();
if (!overflow && len) {
  MyBuffer[len] = '\0';
  Serial.print(MyBuffer);
}
else {
  Serial.print("Overflow!");
}
```

当然ながら、ローカルスコープのバッファメモリを設定し、
それを切断する前に当該スコープを抜けてバッファメモリが開放された場合は、破滅的な結果を招く。

HardwareSerial ブリッジに対しても同様に機能する。

### void setWriteBack ( void(\*CALLBACK)(MultiUART\*) )

実験的実装：
このメソッドでコールバック関数を設定すると、
write() メソッドが1文字送信しおえる度にコールバックが呼び出される。
コールバックの引数には元の MultiUARTオブジェクトが渡される。

この機能は UARTデバイスが都度入力エコーバックを返してくる場合に、
受信バッファが溢れる前に（バッファメモリを拡張することなく）
そのエコーバックを取り出す機会を得るためにある。
特に print、println で定型コマンドを送信し、
同時に受信バッファサイズを超えるエコーバックや応答を受けとらねばならない際に役立つ。

一方で find() や parseInt() といった受信バッファに応答が
貯められていることを前提にした Streamメソッドの動作を、この機能は妨害するだろう。

```c
// UART.write() が実行を終える度に呼び出されるコールバック
// callback(&UART) の形で呼ばれる
void callback (MultiUART* UART) {
  while (UART->available()) {
    // エコーバックを直ちにコンソールへ書き戻す
    // ただしこれは割込タイミングを崩すかもしれない
    // 他のオフラインバッフアヘ転送するといった処理がより適切だろう
    Serial.write(UART->read());
  }
}

// コールバック設定
UART.setWriteBack(callback);

// 1キャラクタずつ転送
while (Serial.available()) {
  UART.write(Serial.read());
}

// 定型文送信
UART.println(F("$PMTK314,0,1,0,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0*28"));

// コールバック解除
UART.setWriteBack();

```

この機能を停止するには、空引数でメソッドを呼ぶ。

コールバックには現在のところ、自動的に this を設定するようなことはない。
this は呼び出される側の名前空間のままである。

HardwareSerial ブリッジに対しても同様に機能する。

### void setThrottle (int16_t CLOCKALIGN)

実験的実装：
基準クロックを増減するためのスロットル値を整数で指定する。
設定単位は CPUクロックで、正数で割込間隔を減少（早く）、負数で割込間隔を増加（遅く）する。
この値はすべての MultiUART が共有するグローバル設定であり、個別に設定することは出来ない。
メソッドを実行すると
停止していた割込処理は再開され、
現在の TIMER はリセットされるため、
無通信時に行うこと。

```c
// 普通は setup()内で直接使用する
void setup (void) {
  MultiUART::setThrottle(1);
  ;
}

// オブジェクトメソッドとしても呼べる
// ただしオブジェクト個別の調整はできない
UART.begin(9600);
UART.setThrottle(1);
```

この機能は精度の低い MCU内臓RC発信器を使用する場合の調整用途である。

HardwareSerial ブリッジに対しては機能しない。

このメソッドは 0.9.4 で実装されたが 0.9.6 で仕様が変更された。

## その他

Stream と Print クラスを継承しているため、以下のメソッドが機能する。

    print() println()

以下のメソッドは使えるはずだが、充分にはテストされていない。

    setTimeout() find() findUntil() readBytes() readBytesUntil()
    readString() readStringUntil() parseInt() parseFloat()

以下のコールバック関数宣言はサポートされない。

    serialEvent()


## 応用

### 基準クロックの変更

ライブラリをインクルードする前に、以下の定数を宣言することで規定の基準クロックを変更できる。

```c
#define MULTIUART_BASEFREQ 19200
#include <MultiUART.h>
```

既定値は 19200 で、その 1/2 が使用可能な最大ボーレートとなる。
従って 38400 を指定すれば 19200bps が使用可能になる。
ただし 19200設定時、CPUが 16Mhz駆動で 45%、
8Mhz駆動で 90% もの CPU稼働時間が割込処理のためだけに消費される点に留意が必要だ。

### 受信バッファサイズの変更

ライブラリをインクルードする前に、以下の定数を宣言することで規定の受信バッファサイズを変更できる。

```c
#define MULTIUART_RX_BUFF_LEN 64
#include <MultiUART.h>
```

既定値は 64で、一般的な UARTデバイスの「受諾」プロンプトを格納するには少なくはないだろう。
指定できる値には制約があり、256以下でかつ2のべき乗数としなければならない。
すなわち指定できる値は実用上、以下の値に限られる。

    256 128 (64 32 16)

この値の変更は SRAMの空き容量とバーターである。
さらに現状では、個別のオブジェクト毎に割り当てを変えることも出来ない。
この値を増やすよりは setRxBuffer() を用いて個別の外部受信バッファを割り当てるほうが望ましい。

この値の既定値は 0.9.3 で変更された。

### 同時受信可能UART数の変更

ライブラリをインクルードする前に、以下の定数を宣言することで規定の同時受信可能なUART数を変更できる。

```c
#define MULTIUART_RX_LISTEN_LEN 4
#include <MultiUART.h>
```

既定値は 4である。

この値を減らすのはともかく、増やすのは賢明ではない。stopListening() を活用すべきである。

### ボーレート生成タイマーの変更

ライブラリをインクルードする前に、以下の定数を宣言することで規定のボーレート生成タイマーを変更できる。

```c
#define MULTIUART_USED_TIMER1   // TIMER1を選択
#define MULTIUART_USED_TIMER2   // TIMER2を選択（不要な方をコメントアウト）
#include <MultiUART.h>
```

いずれも未定義の場合は MULTIUART\_USED\_TIMER1 （TIMER1を選択）である。

TIMER1は16bitカウンタであるため、分周比は 1/1（CPUクロックに同期）である。
これによる生成ボーレートは 16Mhz駆動の場合、19207.68307323bps となる。

しかし実際にはこれより 1%程度高くしないと、多重割込遅延による受信フレームエラーが回避できないことが多い。

setThrottle() が、この設定を動的に調律可能とするため 0.9.4 で追加実装された。

一方、TIMER2は 8bitカウンタであるため分周比は 1/8（8CPUクロック精度）で使用される。
そのぶん微調整は難しい。
0.9.1 ではこちらが規定値であった。

この設定は 0.9.2 で追加された。

## 既知の不具合／制約／課題

- 受信割込、serialEvent() に相当する機能がない。
- 主要な AVR 以外はテストされていない。
- 古い Arduino IDE には対応しない。1.8.5で動作確認。少なくとも C++11 は使えなければならない。
- 英文マニュアルが未整備である。

## 改版履歴

- 0.9.6
  - 機能拡張：HardwareSerial ブリッジ

## 使用許諾

MIT

## 著作表示

朝日薫 / askn
(SenseWay Inc.)
Twitter: [@askn37](https://twitter.com/askn37)
GitHub: https://github.com/askn37
