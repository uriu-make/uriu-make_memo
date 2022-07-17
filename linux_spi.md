SPIを扱うために必要なヘッダファイルは以下の3つです。
```
#include <linux/spi/spidev.h>
#include <fcntl.h>
#include <sys/ioctl.h>
```
/dev/spidev0.0をO_RDWR(読み書き両用)で開く。戻り値はファイルディスクリプタ\
ファイル名は操作するSPIバスに一致するものを選択
```
int fd = open("/dev/spidev0.0", O_RDWR);
```
## SPIの設定
```
//SPIのモードを設定
__u8 mode = SPI_MODE_1;
iocrl(fd, SPI_IOC_WR_MODE, &mode);
ioctl(fd, SPI_IOC_RD_MODE, &mode);

//クロック周波数のデフォルトの最大値を設定
__u32 freq = 1000000;
ioctl(fd, SPI_IOC_WR_MAX_SPEED_HZ, &freq);
ioctl(fd, SPI_IOC_RD_MAX_SPEED_HZ, &freq);

//データをLSBから読み取るように指定(任意)
__u8 lsb_first = SPI_LSB_FIRST;
ioctl(fd, SPI_IOC_WR_LSB_FIRST, &lsb_first);
ioctl(fd, SPI_IOC_RD_LSB_FIRST, &lsb_first);

//1ワードのビット数を変更(使用可能な値のみ有効)
__u8 bits_par_word = 8;
ioctl(fd, SPI_IOC_RD_BITS_PER_WORD, &bits_par_word);
ioctl(fd, SPI_IOC_WR_BITS_PER_WORD, &bits_par_word);
```
## 送受信
```
struct spi_ioc_transfer arg[N];
ioctl(spi_fd, SPI_IOC_MESSAGE(N), arg);
```
Nは転送するstruct spi_ioc_transferの数。\
0の場合は設定は変更されない。
| spi_ioc_transferのメンバ変数 | 説明                                                                                           |
| ---------------------------- | ---------------------------------------------------------------------------------------------- |
| __u64 tx_buf                 | 送信するデータを代入した変数のアドレスを代入。__u64にキャストすることを推奨                    |
| __u64 rx_buf;                | 受信舌データを代入する変数のアドレスを代入。__u64にキャストすることを推奨                      |
| __u32 len;                   | 送受信するデータの総バイト数を代入                                                             |
| __u32 speed_hz;              | 通信速度を代入(Hz)                                                                             |
| __u16 delay_usecs;           | 次のデータ転送の待ち時間を代入(μS)                                                             |
| __u8 bits_per_word;          | 1ワードのビット長を上書き                                                                      |
| __u8 cs_change;              | Trueのとき、次の転送の間にデバイスの選択の解除をする.                                          |
| __u8 tx_nbits;               | 書き込みで使用するビット数                                                                     |
| __u8 rx_nbits;               | 読み込みで使用するビット数                                                                     |
| __u8 word_delay_usecs;       | 1回の転送でワード間の時間を設定。SPIコントローラーで明示的にサポートされない場合は無視される。 |
| __u8 pad;                    | 不明                                                                                           |