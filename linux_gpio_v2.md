ここでは、C/C++で/dev/gpiochipNからアクセスしてGPIOを操作する方法について書きます。\
Linux 4.8以降では/sys/class/gpioを使用したアクセスは推奨されていません。\
ここでの方法はABI v2の一部で、環境によっては動作しない場合が考えられます。\
読み込むヘッダファイルは
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
/dev/gpiochipNをO_RDWR(読み書き両用)で開く。戻り値はファイルハンドル
### 入出力モードの設定
```
struct gpio_v2_line_request req;
ioctl(fd, GPIO_V2_GET_LINE_IOCTL, &req);
```
| struct gpio_v2_line_requestのメンバ変数 | 説明                                                                                                                                                                                                                                                                                                                |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| __u32 fd;                               | 成功したとき、GPIOラインのハンドルが代入される。0か負の数のときは失敗                                                                                                                                                                                                                                               |
| __u32 padding\[5\];                     | 将来の使用のために予約されている配列。0で埋める必要がある。                                                                                                                                                                                                                                                         |
| __u32 event_buffer_size;                | バッファリングする必要のあるラインイベントの推奨最小数。エッジ検出を有効にしている場合に使用される。<br>バッファサイズはカーネル側で設定される場合があるため、この値よりも大きいバッファを割り当てることもあれば、より小さいバッファを割り当てることがあることに注意が必要。0の場合はnum_lines*16が割り当てられる。 |
| __u32 num_lines;                        | GPIOラインで使用するGPIOの総数を代入。offsetsで有効なフィールドの数                                                                                                                                                                                                                                                 |
| struct gpio_v2_line_config config;      | GPIOラインの構成。詳細は後述                                                                                                                                                                                                                                                                                        |
| char consumer\[GPIO_MAX_NAME_SIZE\];    | GPIOラインにつけるラベル。必要に応じてつける。                                                                                                                                                                                                                                                                      |
| __u32 offsets\[GPIO_V2_LINES_MAX\];     | GPIOラインで使用するGPIOのピン番号                                                                                                                                                                                                                                                                                  |

| struct gpio_v2_line_configのメンバ変数                                    | 説明                                                                                                                                                    |
| ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| __aligned_u64 flags;                                                      | GPIOラインのフラグ。<br>enum gpio_v2_line_attr_idから選択<br>複数の属性(入力でプルアップなど)が必要な場合は論理和(OR)をすることで設定可能               |
| __u32 num_attrs;                                                          | gpio_v2_line_config_attributで有効なフィールドの数                                                                                                      |
| __u32 padding\[5\];                                                       | 将来の使用のために予約されている配列。0で埋める必要がある。                                                                                             |
| struct gpio_v2_line_config_attribute attrs\[GPIO_V2_LINE_NUM_ATTRS_MAX\]; | GPIOラインに関連付けられた構成属性。<br>属性が1つの行に複数回関連付けられている場合は、最も低いインデックスで宣言されたものが優先される。<br>詳細は後述 |

| enum gpio_v2_line_flagの列挙子   | 説明                                 |
| -------------------------------- | ------------------------------------ |
| GPIO_V2_LINE_FLAG_USED           | 選択されたGPIOラインはリクエスト不可 |
| GPIO_V2_LINE_FLAG_ACTIVE_LOW     | LOWをアクティブな状態にする          |
| GPIO_V2_LINE_FLAG_INPUT          | GPIOラインを入力に設定               |
| GPIO_V2_LINE_FLAG_OUTPUT         | GPIOラインを出力に設定               |
| GPIO_V2_LINE_FLAG_EDGE_RISING    | 非アクティブ->アクティブの変化を検出 |
| GPIO_V2_LINE_FLAG_EDGE_FALLING   | アクティブ->非アクティブの変化を検出 |
| GPIO_V2_LINE_FLAG_OPEN_DRAIN     | 未検証                               |
| GPIO_V2_LINE_FLAG_OPEN_SOURCE    | 未検証                               |
| GPIO_V2_LINE_FLAG_BIAS_PULL_UP   | GPIOラインをプルアップする           |
| GPIO_V2_LINE_FLAG_BIAS_PULL_DOWN | GPIOラインをプルダウンする           |
| GPIO_V2_LINE_FLAG_BIAS_DISABLED  | GPIOラインにバイアスを設けない       |

| struct gpio_v2_line_config_attributeのメンバ変数 | 説明                                                                                                                                                 |
| ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| struct gpio_v2_line_attribute attr;              | 属性の構成<br>詳細は後述                                                                                                                             |
| __aligned_u64 mask;                              | 構成を有効にするGPIOのマスクのビットマップ。<br>struct gpio_v2_line_request.offsets\[0\]がLSBに対応し、以降offsetsに追加した順に各ビットが対応する。 |

| struct gpio_v2_line_attributeのメンバ変数 | 説明                                                                                                                                                                                                                                                                            |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| __u32 id;                                 | 設定する属性の識別子<br>enum gpio_v2_line_attr_idから選択                                                                                                                                                                                                                       |
| __u32 padding;                            | 将来の使用のために予約されている変数。0で埋める必要がある。                                                                                                                                                                                                                     |
| __aligned_u64 flags;                      | idがGPIO_V2_LINE_ATTR_ID_FLAGSの場合、GPIOラインのフラグが設定されます。<br>enum gpio_v2_line_attr_idから選択でき、複数の属性(入力でプルアップなど)が必要な場合は論理和(OR)をすることで設定可能<br>これは関連するstruct gpio_v2_line_configのデフォルトのフラグを上書きします。 |
| __aligned_u64 values;                     | idがGPIO_V2_LINE_ATTR_ID_OUTPUT_VALUESの場合に設定されるビットマップ。<br>struct gpio_v2_line_request.offsets\[0\]がLSBに対応し、以降offsetsに追加した順に各ビットが対応する。<br>アクティブは1,非アクティブは0                                                                 |
| __u32 debounce_period_us;                 | idがGPIO_V2_LINE_ATTR_ID_DEBOUNCEの場合に設定されるデバウンス時間。<br>機械式スイッチのデバウンスやチャタリングを抑えるために使用                                                                                                                                               |

| enum gpio_v2_line_attr_idの列挙子  | 説明                           |
| ---------------------------------- | ------------------------------ |
| GPIO_V2_LINE_ATTR_ID_FLAGS         | GPIOラインのフラグを設定       |
| GPIO_V2_LINE_ATTR_ID_OUTPUT_VALUES | GPIOラインのビットマップを設定 |
| GPIO_V2_LINE_ATTR_ID_DEBOUNCE      | デバウンスの時間を設定         |

## 入出力
``````
struct gpio_v2_line_values values;
//出力
ioctl(req.fd, GPIO_V2_LINE_SET_VALUES_IOCTL, &values);
//入力
ioctl(req.fd, GPIO_V2_LINE_GET_VALUES_IOCTL, &values);
``````
value.maskに入力を含んだ状態で出力を行った場合、出力されません。
| struct gpio_v2_line_valuesのメンバ変数 | 説明                                                    |
| -------------------------------------- | ------------------------------------------------------- |
| __aligned_u64 bits;                    | GPIOラインのビットマップ。アクティブは1,非アクティブは0 |
| __aligned_u64 mask;                    | 取得または設定する行を識別するビットマップ              |

## イベントの検出
```
struct gpio_v2_line_event event;
read(req.fd, &event, sizeof(event));
```
| struct gpio_v2_line_eventのメンバ変数 | 説明                                                                                         |
| ------------------------------------- | -------------------------------------------------------------------------------------------- |
| __aligned_u64 timestamp_ns;           | イベント発生時間の最適な見積もり(ナノ秒)<br>CLOCK_MONOTONICが読み取られる。                  |
| __u32 id;                             | enum gpio_v2_line_event_idで定義される検出されたイベントの識別子                             |
| __u32 offset;                         | イベントが発生したGPIOのピン番号                                                             |
| __u32 seqno;                          | すべてのGPIOラインのイベントに対してのこのGPIOラインにおける発生したイベントのシーケンス番号 |
| __u32 line_seqno;                     | GPIOラインにおける発生したイベントのシーケンス番号                                           |
| __u32 padding\[6\];                   | 将来の使用のために予約済み                                                                   |

| enum gpio_v2_line_event_idの列挙子 | 説明                           |
| ---------------------------------- | ------------------------------ |
| GPIO_V2_LINE_EVENT_RISING_EDGE     | 非アクティブ->アクティブの変化 |
| GPIO_V2_LINE_EVENT_FALLING_EDGE    | アクティブ->非アクティブの変化 |

## 入出力モードの再設定
```
struct gpio_v2_line_config config;
ioctl(req.fd, GPIO_V2_LINE_SET_CONFIG_IOCTL, &config);
```
GPIOラインの入出力、イベント検出の設定を上書きする。\
ピンは追加されない
