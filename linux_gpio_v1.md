ここでは、C/C++で/dev/gpiochipNからアクセスしてGPIOを操作する方法について書きます。\
Linux 4.8以降では/sys/class/gpioを使用したアクセスは推奨されていません。\
また、ここでの方法はABI v1の一部であり、ABI v2が使用可能な環境の場合、非推奨です。\
読み込むライブラリは
```
#include <linux/gpio.h>
#include <fcntl.h>
#include <sys/ioctl.h>
```
の3種類が最低限必要です。
## 初期化
```
int fd = open("/dev/gpiochipN", O_RDWR);
```
/dev/gpiochipNをO_RDWRで開く。戻り値はファイルハンドル
### 入出力モードの設定
```
struct gpiohandle_request request;
ioctl(fd, GPIO_GET_LINEHANDLE_IOCTL, &request);
```
|struct gpiohandle_requestのメンバ変数|使用するGPIOの初期設定|
|-|-|
|__u32 lineoffsets\[GPIOHANDLES_MAX\];|使用するGPIOのピン番号を代入する<br>複数ピンをまとめて設定可能|
|__u32 flags;|GPIOの入出力モードを設定<br>下記参照|
|__u8 default_values\[GPIOHANDLES_MAX\];|GPIOを出力に設定したときの初期値<br>複数をまとめて操作している場合は`lineoffsets`に代入した順|
|char consumer_label\[GPIO_MAX_NAME_SIZE\];|GPIOラインにつける名前|
|__u32 lines;|GPIOラインで使用する入出力の数|
|int fd;|GPIOラインの初期化が完了したときに発行されるファイルハンドル<br>0または負の値のときはエラー|

|flagsで代入できる定数|説明|
|---|---|
|#define GPIOHANDLE_REQUEST_INPUT|GPIOを入力モードに設定|
|#define GPIOHANDLE_REQUEST_OUTPUT|GPIOを出力モードに設定|
|#define GPIOHANDLE_REQUEST_ACTIVE_LOW|LOWをアクティブな状態にする|
|#define GPIOHANDLE_REQUEST_OPEN_DRAIN|未検証|
|#define GPIOHANDLE_REQUEST_OPEN_SOURCE|未検証|
|#define GPIOHANDLE_REQUEST_BIAS_PULL_UP|プルアップ|
|#define GPIOHANDLE_REQUEST_BIAS_PULL_DOWN|プルダウン|
|#define GPIOHANDLE_REQUEST_BIAS_DISABLE|バイアスを有効にしない|
### イベントの設定
GPIOピンの状態の変化を検出する設定
```
struct gpioevent_request &event_request;
ioctl(fd, GPIO_GET_LINEEVENT_IOCTL, &event_request);
```
|struct gpioevent_requestのメンバ変数|説明|
|-|-|
|__u32 lineoffset;|使用するGPIOのピン番号を代入|
|__u32 handleflags;|GPIOラインのハンドルフラグを指定|
|__u32 eventflags;|監視するGPIOの状態の変化の指定|
|char consumer_label\[GPIO_MAX_NAME_SIZE\];|GPIOラインにつける名前|
|int fd;|GPIOラインの初期化が完了したときに発行されるファイルハンドル<br>0または負の値のときはエラー|

|handleflagsで代入できる定数|説明|
|-|-|
|#define GPIOEVENT_REQUEST_RISING_EDGE|GPIOがLow->Highの変化をしたとき反応する|
|#define GPIOEVENT_REQUEST_FALLING_EDGE|GPIOがHigh->Lowの変化をしたとき反応する|
|#define GPIOEVENT_REQUEST_BOTH_EDGES|GPIOがLow->High、High->Lowのどちらでも反応する|
## 入出力
```
struct gpiohandle_data value;
//output
ioctl(request.fd, GPIOHANDLE_SET_LINE_VALUES_IOCTL, &value);

//input
ioctl(request.fd, GPIOHANDLE_GET_LINE_VALUES_IOCTL, &value);
```
|struct gpiohandle_dataのメンバ変数|GPIOの情報を扱うハンドル|
|-|-|
|__u8 values\[GPIOHANDLES_MAX\]|GPIOの値<br>出力は変数への代入、入力は変数の読み取りで扱える。<br>複数をまとめて操作している場合は`lineoffsets`に代入した順|
## イベントの検出
```
struct gpioevent_data event_data;
read(event_request.fd, &event_data, sizeof(event_data));
```
|struct gpioevent_dataのメンバ変数|説明|
|-|-|
|__u64 timestamp;|発生したイベントの最も近いと推測される時間(ナノ秒)|
|__u32 id;|発生したイベントの種類|

|イベントの種類|説明|
|-|-|
|#define GPIOEVENT_EVENT_RISING_EDGE|GPIOがLow->Highの変化のとき|
|#define GPIOEVENT_EVENT_FALLING_EDGE|GPIOがHigh->Lowの変化のとき|