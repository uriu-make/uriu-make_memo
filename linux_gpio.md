読み込むライブラリは
```
#include <linux/gpio.h>
#include <fcntl.h>
#include <sys/ioctl.h>
```
の3種類が最低限必要
## 初期化
```
int fd = open("/dev/gpiochip*", O_RDWR);
```
/dev/gpiochip*をO_RDWRで開く。戻り値はファイルハンドル
### 入出力モードの設定
```
ioctl(int fd, GPIO_GET_LINEHANDLE_IOCTL, struct gpiohandle_request &req);
```
ここでの`fd`は/dev/gpiochip*を開いたときに発行されるファイルハンドル
|struct gpiohandle_requestのメンバ変数|使用するGPIOの初期設定|
|-|-|
|__u32 lineoffsets\[GPIOHANDLES_MAX\];|使用するGPIOのBCM番号を代入する<br>複数ピンをまとめて設定可能|
|__u32 flags;|GPIOの入出力モードを設定<br>下記参照|
|__u8 default_values\[GPIOHANDLES_MAX\];|GPIOを出力に設定したときの初期値<br>複数をまとめて操作している場合は`lineoffsets`に代入した順|
|char consumer_label\[GPIO_MAX_NAME_SIZE\];|GPIOラインにつける名前|
|__u32 lines;|GPIOラインで使用する入出力の数|
|int fd;|GPIOラインの初期化が完了したときに発行されるファイルハンドル<br>0または負の値のときはエラー|

|flagsで代入できる定数|説明|
|---|---|
|#define GPIOHANDLE_REQUEST_INPUT|GPIOを入力モードに設定|
|#define GPIOHANDLE_REQUEST_OUTPUT|GPIOを出力モードに設定|
|#define GPIOHANDLE_REQUEST_ACTIVE_LOW|未検証|
|#define GPIOHANDLE_REQUEST_OPEN_DRAIN|未検証|
|#define GPIOHANDLE_REQUEST_OPEN_SOURCE|未検証|
|#define GPIOHANDLE_REQUEST_BIAS_PULL_UP|プルアップ|
|#define GPIOHANDLE_REQUEST_BIAS_PULL_DOWN|プルダウン|
|#define GPIOHANDLE_REQUEST_BIAS_DISABLE|未検証|
### イベントの設定
GPIOピンの状態の変化を検出する設定
```
ioctl(fd, GPIO_GET_LINEEVENT_IOCTL, struct gpioevent_request &req);
```
ここでの`fd`は/dev/gpiochip*を開いたときに発行されるファイルハンドル
|struct gpioevent_requestのメンバ変数|説明|
|-|-|
|__u32 lineoffset;|使用するGPIOのBCM番号を代入|
|__u32 handleflags;|GPIOラインのハンドルフラグを指定|
|__u32 eventflags;|監視するGPIOの状態の変化の指定|
|char consumer_label\[GPIO_MAX_NAME_SIZE\];|GPIOラインにつける名前|
|int fd;|GPIOラインの初期化が完了したときに発行されるファイルハンドル<br>0または負の値のときはエラー|
|__u32 handleflags|説明|
|-|-|
|#define GPIOEVENT_REQUEST_RISING_EDGE|GPIOがLow->Highの変化をしたとき反応する|
|#define GPIOEVENT_REQUEST_FALLING_EDGE|GPIOがHigh->Lowの変化をしたとき反応する|
|#define GPIOEVENT_REQUEST_BOTH_EDGES|GPIOがLow->High、High->Lowのどちらでも反応する|
## 入出力
```
//output
ioctl(int fd, GPIOHANDLE_SET_LINE_VALUES_IOCTL, struct gpiohandle_data *value);

//input
ioctl(int fd, GPIOHANDLE_GET_LINE_VALUES_IOCTL, struct *gpiohandle_data *value);
```
ここでの`int fd`はGPIOの初期化で発行されたgpiohandle_requestのメンバ変数fdの値
|struct gpiohandle_dataのメンバ変数|GPIOの情報を扱うハンドル|
|-|-|
|__u8 values\[GPIOHANDLES_MAX\]|GPIOの値<br>出力は変数への代入、入力は変数の読み取りで扱える。<br>複数をまとめて操作している場合は`lineoffsets`に代入した順|
## イベントの検出
```
read(int fd, struct gpioevent_data value, sizeof(struct gpioevent_data));
```
ここでの`fd`はイベントの設定で発行されるgpioevent_requestのメンバ変数fdの値
|struct gpioevent_dataのメンバ変数|説明|
|-|-|
|__u64 timestamp;|発生したイベントの最も近いと推測される時間(ナノ秒)|
|__u32 id;|発生したイベントの種類|

|イベントの種類|説明|
|-|-|
|#define GPIOEVENT_EVENT_RISING_EDGE|GPIOがLow->Highの変化のとき|
|#define GPIOEVENT_EVENT_FALLING_EDGE|GPIOがHigh->Lowの変化のとき|