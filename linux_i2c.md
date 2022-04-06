I2Cを扱うために必要なヘッダファイルは以下の４つです。
```
#include <linux/i2c-dev.h>
#include <linux/i2c.h>
#include <fcntl.h>
#include <sys/ioctl.h>
```
/dev/i2c-1をO_RDWR(読み書き両用)で開く。戻り値はファイルディスクリプタ\
開くファイル名は使用するI2Cラインに対応したものを選択する必要があります。
```
__u16 address = 0x3c;
int fd = open("/dev/i2c-1", O_RDWR);
int device_fd = ioctl(fd, I2C_SLAVE, address);
```
### 単方向通信
書き込み/読み込みのみの場合は
```
char data[2]:
//読み込み
read(device_fd, data, sizeof(data));
//書き込み
write(device_fd, data, sizeof(data));
```
でも動作します。
### 双方向通信など
Repeated Start Conditionを使用した通信は以下の方法で行います。
```
struct i2c_msg args[2];
struct i2c_rdwr_ioctl_data message;
__u8 reg = 0x00;
__u8 data[3];

args[0].addr = address;
args[0].flags = 0; //書き込み
args[0].len = sizeof(reg);
args[0].buf = &reg;
args[1].addr = address;
args[1].flags = I2C_M_RD; //読み込み
args[1].len = sizeof(data);
args[1].buf = data;

message.msgs = args;
message.nmsgs = 2;

ioctl(fd, I2C_RDWR, &message);
```
Repeated Start ConditionはStopコンディションの代わりにStartコンディションにすることで、再度通信を行う方法。\
Stopコンディションが入ると受信した値を維持しないデバイスなどで通信を終了することなく送信/受信を切り替えたい場合やマルチマスタで通信の中断を防ぐために使われます。
|struct i2c_msgのメンバ変数|説明|
|-|-|
|__u16 addr;|スレーブデバイスのアドレス|
|__u16 flags;|読み込み/書き込みなどのモードの設定|
|__u16 len;|データのバイト数|
|__u8 *buf;|送受信するデータの配列|

flagsに0を代入した場合は書き込みの動作になり、I2C_M_RDを代入した場合は読み取りの動作になる。

|struct i2c_rdwr_ioctl_dataのメンバ変数|説明|
|-|-|
|struct i2c_msg *msgs;|struct i2c_msgのポインタ|
|__u32 nmsgs;|struct i2c_msgの要素数|