---
theme: default
title: MicroPythonとRaspberry Pi Pico Wで始めるワイヤレス通信
info: |
  ## PyCon JP 2024 発表資料
  MicroPythonとRaspberry Pi Pico Wで始めるワイヤレス通信

  中村草介(Sousuke NAKAMURA)
drawings:
  persist: false
transition: slide-left
mdc: true
layout: cover
---

# PyCon JP 2024 発表資料
## MicroPythonとRaspberry Pi Pico Wで始めるワイヤレス通信

中村草介(Sousuke NAKAMURA)

---
layout: center
---

## はじめに

- 15分の枠での発表です
- 概要のあと、5分のLTが二つ続くイメージで用意しています

---

## 自己紹介

- 中村草介
- Raspberry Piもくもく会スタッフ
- Raspberry Piもくもく会はポスターセッションでも出展しています

<img src='/75330.jpeg' align='right' width='200px'/>


---

## 自己紹介(お仕事の紹介)

- 株式会社アーバンエックステクノロジーズ
  - 道路など都市インフラの管理をソフトウェアでサポートするサービスを提供しています
- ソフトウェアエンジニアのみなさま、絶賛募集中です！

<img src='https://urbanx-tech.com/wp-content/uploads/2022/08/cropped-Logo@3x.png' align='right' width='200px'/>


---
layout: two-cols-header
---

## Raspberry Pi Pico Wとは

::left::

- Raspberry Pi PicoにWi-Fi、Bluetooth機能を追加したボードです
  - Linuxが動作するRaspberry Piとは違います
  - WiFi、Bluetoothを担うInfineon CYW43439がSPIでRP2040に接続されています
    - WiFi 802.11n(2.4 GHz Wi-Fi 4)
    - Bluetooth® 5.2(BDR(1 Mbps)/EDR(2/3 Mbps)/Bluetooth® LE)


<https://www.raspberrypi.com/documentation/microcontrollers/pico-series.html>

::right::

<img src='/IMG_7755.jpg' width='300px' align='right'/>


---

## MicroPythonとは

- Python3のサブセットで、マイクロコントローラー向けに最適化された言語です
- 使い慣れたPythonと標準ライブラリがある程度使えます

<https://micropython.org>

---

## ライブラリ一覧
- 添付されているライブラリの一覧
```python
>>> help('modules')
__main__          asyncio/__init__  hashlib           rp2
_asyncio          asyncio/core      heapq             select
_boot             asyncio/event     io                socket
_boot_fat         asyncio/funcs     json              ssl
_onewire          asyncio/lock      lwip              struct
_rp2              asyncio/stream    machine           sys
_thread           binascii          math              time
_webrepl          bluetooth         micropython       tls
aioble/__init__   builtins          mip/__init__      uasyncio
aioble/central    cmath             neopixel          uctypes
aioble/client     collections       network           urequests
aioble/core       cryptolib         ntptime           vfs
aioble/device     deflate           onewire           webrepl
aioble/l2cap      dht               os                webrepl_setup
aioble/peripheral ds18x20           platform          websocket
aioble/security   errno             random
aioble/server     framebuf          re
array             gc                requests/__init__
Plus any modules on the filesystem
```

---

## MicroPythonのインストール

- uf2ファイルをダウンロード <https://micropython.org/download/RPI_PICO_W/>
- Rasppeberry Pi Pico WのBOOTSELボタンを押しながらUSBケーブルをコンピューターに接続
- USBストレージとして認識されるので、MicroPythonのuf2ファイルをコピーします
- コピーが終わると自動的に再起動(アンマウント)されます

---

## mpremoteを使って動作確認

- Raspberry Pi Pico W上ではMicroPythonのREPLが起動しているのでシリアルコンソールで接続することができます
- ここではmpremoteモジュールを使って接続することにします
  - https://pypi.org/project/mpremote/

```shell
$ pip install mpremote
$ mpremote connect list
/dev/cu.usbmodem101 e661385283776133 2e8a:0005 MicroPython Board in FS mode
$ mpremote connect /dev/cu.usbmodem101
Connected to MicroPython at /dev/cu.usbmodem101
Use Ctrl-] or Ctrl-x to exit this shell

>>> import sys
>>> sys.implementation
(name='micropython', version=(1, 23, 0, ''), _machine='Raspberry Pi Pico W with RP2040', _mpy=4870)
>>>
```


---

## 開発環境の準備

- RaspberryPiの公式ドキュメントではThonny (<https://thonny.org>) がお勧めされています
- <https://projects.raspberrypi.org/en/projects/getting-started-with-the-pico>
- ファイル(ローカル、デバイス)、エディター、REPL、デバッグ用のペインが揃っています

<img src='/thonny.png' align='right' width='400px'/>



---
layout: two-cols-header
---

## まずはhello worldしましょう

- hello worldに相当するLEDの点滅
- Arduinoのサンプルコードと同じような流れでLEDを点滅させることができます

::left::
- Arduinoのチュートリアルにあるサンプルコード

```cpp
void setup() {
   pinMode(LED_BUILTIN, OUTPUT);
}

void loop() {
   digitalWrite(LED_BUILTIN, HIGH);
   delay(1000);
   digitalWrite(LED_BUILTIN, LOW);
   delay(1000);
}
```

<https://docs.arduino.cc/tutorials/uno-rev3/Blink/>


::right::

- Micro Python

```python
import machine
import time

led = machine.Pin('LED', machine.Pin.OUT)

while True:
    led.value(1)
    time.sleep(1)
    led.value(0)
    time.sleep(1)
```

---

## MicroPythonでのLチカ

- PythonなのでTimerオブジェクトのコールバックにlambdaを使ってシンプルに書くこともできます

```python
import machine
led = machine.Pin('LED', machine.Pin.OUT)
timer = machine.Timer()
timer.init(freq=2.5, mode=machine.Timer.PERIODIC, callback=lambda _: led.toggle())
```

## 内蔵の温度センサの値を測ってみましょう

- 以下の計算をすることでCPUの温度が求められます(RP2040のデータシートに記載があります)
  - `T = 27 - (ADC_voltage - 0.706)/0.001721`
  - https://datasheets.raspberrypi.com/rp2040/rp2040-datasheet.pdf

```
import machine
import time

temp_adc = machine.ADC(4)

def get_temperature():
    v = (3.3/65535) * temp_adc.read_u16()
    t = 27 - (v - 0.706)/0.001721
    return t

while True:
    print(get_temperature())
    time.sleep(1)
```

---
layout: two-cols-header
---
## 2 wifi

- wifi接続はnetworkモジュールを使います
- network.STA_IF(子機として既存のネットワークに接続する)とnetwork.AP_IF(親機として接続を受け付ける)があります

::left::
```python
import network
import rp2
import time


rp2.country('JP')

wlan = network.WLAN(network.STA_IF)
wlan.active(True)
wlan.connect('SSID', 'password')

while not (wlan.isconnected() and wlan.status() == network.STAT_GOT_IP):
   print('Waiting to connect:')
   time.sleep(1)

print(wlan.ifconfig())
```
::right::
```python
import network
import rp2
import time


rp2.country('JP')

wlan = network.WLAN(network.AP_IF)
wlan.config(ssid='raspberry_pi_pico_w', key='password')
wlan.ifconfig(('192.168.31.4', '255.255.255.0', '192.168.31.1', '8.8.8.8'))
wlan.active(True)

while wlan.active() == False:
   print('Waiting to active')
   time.sleep(1)

print('ready')

```

---

## HTTPリクエストを送信する

- requestsライブラリが利用可能

```python
import json
import urequests

# 略

WEATHER_API_URL = 'https://api.open-meteo.com/v1/forecast?latitude=35.680959106959&longitude=139.76730676352&current=temperature_2m,wind_speed_10m'

response = urequests.get(WEATHER_API_URL)
response_body = json.loads(response.text)

for key in ['temperature_2m', 'wind_speed_10m']:
  print('key: ' + str(response_body['current'][key]))

```

---

## HTTPサーバーを立てたい

- 添付されている標準モジュールの一覧
  - <https://docs.micropython.org/en/latest/genrst/index.html>

```
array
builtins
json
os
random
struct
sys
```

- urllibモジュールがないのでurllib.parseなど便利なモジュールが使えない
- httpモジュールがないのでhttp.serverモジュールもない

---

## HTTPサーバーを立てたいので試しに書いてみました

- socketやasyncioはあるのでパース周りが用意できればある程度のものは用意できそう
- Raspberry Pi公式のドキュメントなどではsocketもしくはasyncで待ち受けて`/led/on`や`/led/off`が含まれていればLEDを点けたり消したりする方法が紹介されています
- でももう少し先にいきたい気持ちになったので勢いだけで作ってみていました

```python {monaco} {maxHeight:'400px'}
import asyncio


class WebServer:
    def __init__(self, host='0.0.0.0', port=80, handlers={}):
        self.host = host
        self.port = port
        self.handlers = handlers

    def default_handler(self, method, path, request_header, query_dict={}, request_body=None):
        return b'HTTP/1.0 404 Not Found\r\n\r\nNot Found'

    async def read_request(self, request):
        return (await request.readline()).decode('ascii').strip('\r\n')

    def parse_request_line(self, request_line):
        method, path, version = request_line.split(' ')
        return method, path, version

    def parse_header_line(self, header_line):
        return header_line.split(': ', 1)

    def parse_request_header_to_dict(self, header_list):
        header_dict = {}
        for header_line in header_list:
            key, value = self.parse_header_line(header_line)
            print(f'key: {key}, value: {value}')
            header_dict.update({key: value})
        return header_dict

    def parse_request_path(self, path):
        query_dict = {}
        if '?' in path:
            path, query_string = path.split('?', 1)
            for query_pair in query_string.split('&'):
                key, value = query_pair.split('=', 1)
                query_dict.update({key: value})
        return path, query_dict

    async def read_header(self, request):
        header_list = []
        while True:
            header_line = await self.read_request(request)
            if header_line == '':
                break
            header_list.append(header_line)
        return header_list

    async def read_request_body(self, request, content_length, charset='utf-8'):
        return (await request.read(content_length)).decode(charset)

    async def dispatch(self, request, response):
        headers = await self.read_header(request)
        method, path, version = self.parse_request_line(headers.pop(0))
        request_header = self.parse_request_header_to_dict(headers)
        path, query_dict = self.parse_request_path(path)

        if 'Content-Length' in request_header and int(request_header.get('Content-Length', 0)) > 0:
            request_body = await self.read_request_body(request, int(request_header.get('Content-Length', 0)))

        handler = self.handlers.get(path, self.default_handler)
        response = handler(method, path, request_header, query_dict=query_dict, request_body=request_body)

        response.write(response)
        response.close()
        await response.wait_closed()

    async def serve(self):
        await asyncio.start_server(self.dispatch, self.host, self.port)

web_server = WebServer(host='0.0.0.0', port=1080)
def main():
    loop = asyncio.new_event_loop()
    # loop.set_debug(True)
    loop.create_task(web_server.serve())
    loop.run_forever()

main()

```

---

## 3. Bluetooth
- 以下二つのモジュールが提供されています
  - bluetoothモジュール
    - https://micropython-docs-ja.readthedocs.io/ja/latest/library/bluetooth.html
  - aiobleモジュール
    - <https://github.com/micropython/micropython-lib/tree/master/micropython/bluetooth/aioble>
- 今回はより高レベルなラッパーとして使いやすいaiobleモジュールを使います

---

## aiobleモジュール

- 以下のBluetooth LEで使われる一通りの機能を持っています
  - GAP
    - Broadcaster(どのようなサービスを提供しているかを告知)
    - Observer(ブロードキャストしているCentralを探す)
    - Central(ブロードキャストする側のデバイス/サーバーに相当)
    - Peripheral(Centralへ接続しにいく側のデバイス/クライアントに相当)
  - GATT
    - GATT Server
    - GATT Client
  - L2CAP
    - L2CAP
- GAP(接続されるまでの処理)とGATT Server/Client(接続されてからの処理)について紹介したいです

---

## まずはObserver BLEデバイスのスキャンをしてみます

```python
import aioble
import asyncio


async def ble_task():
   async with aioble.scan(duration_ms=5000) as scanner:
      async for result in scanner:
            print(result, result.name(), result.rssi, result.services())

asyncio.run(ble_task())
```

---

## Broadcaster aiobleを使って温度計サービスのブロードキャストをしてみます

```python
import aioble
import asyncio
import bluetooth


# https://www.bluetooth.com/specifications/assigned-numbers/
_ENV_SENSE_UUID = bluetooth.UUID(0x181A) # Environmental Sensing Service 0x181A Environmental Sensing Service
_ENV_SENSE_TEMP_UUID = bluetooth.UUID(0x2A6E) # Temperature characteristic 0x2A6E Temperature
_GENERIC_THERMOMETER = const(0x0300) # Generic Thermometer appearance 0x00C 0x0300 to 0x033F Thermometer

_ADV_INTERVAL_US = const(250_000)

temp_service = aioble.Service(_ENV_SENSE_UUID)

aioble.register_services(temp_service)

async def instance1_task():
   while True:
      async with await aioble.advertise(interval_us=_ADV_INTERVAL_US, name="temperature sensor", services=[_ENV_SENSE_UUID], appearance=_GENERIC_THERMOMETER, manufacturer=(0xabcd, b"1234")) as connection:
         print("Connection from", connection.device)
         await connection.disconnected(timeout_ms=None)

asyncio.run(instance1_task())
```


---

## Broadcaster スマートフォンなどから確認してみます

<img src='/lightblue_1-3.jpeg' width='150px' float='right'/>
<img src='/lightblue_1-1.jpeg' width='150px' float='right'/>


- LightBlueでアドバタイズができていることを確認できます
  - `name="temperature sensor"`
  - `manufacturer=(0xabcd, b"1234")`
- それぞれの値がアプリ上で確認できています
- <https://punchthrough.com/lightblue/>



---

## Central/GATT Server 温度を自動で更新してみる

- ドキュメント
- <https://github.com/micropython/micropython-lib/blob/master/micropython/bluetooth/aioble/README.md>

```python
import asyncio
import aioble
import bluetooth
import machine

import struct

# https://www.bluetooth.com/specifications/assigned-numbers/
_ENV_SENSE_UUID = bluetooth.UUID(0x181A) # Environmental Sensing Service 0x181A Environmental Sensing Service
_ENV_SENSE_TEMP_UUID = bluetooth.UUID(0x2A6E) # Temperature characteristic 0x2A6E Temperature
_GENERIC_THERMOMETER = const(0x0300) # Generic Thermometer appearance 0x00C 0x0300 to 0x033F Thermometer

_ADV_INTERVAL_US = const(250_000)

# --- ここまでおなじ ---

# GATTサーバーの登録
temp_service = aioble.Service(_ENV_SENSE_UUID)
temp_characteristic = aioble.Characteristic(temp_service, _ENV_SENSE_TEMP_UUID, read=True, notify=True)
aioble.register_services(temp_service)

temp_adc = machine.ADC(4)

def _encode_temperature(temp_deg_c):
    return struct.pack("<h", int(temp_deg_c * 100))

# GATTサーバーとしてCPUの温度を1秒ごとに更新する
async def sensor_task():
    while True:
        v = (3.3/65535) * temp_adc.read_u16()
        t = 27 - (v - 0.706)/0.001721
        temp_characteristic.write(_encode_temperature(t), send_update=True)
        await asyncio.sleep_ms(1_000)

async def peripheral_task():
    while True:
        async with await aioble.advertise(interval_us=_ADV_INTERVAL_US, name="temperature sensor", services=[_ENV_SENSE_UUID], appearance=_GENERIC_THERMOMETER,) as connection:
            print("Connection from", connection.device)
            await connection.disconnected(timeout_ms=None)

async def main():
    task_sensor_update = asyncio.create_task(sensor_task())
    task_peripheral = asyncio.create_task(peripheral_task())
    await asyncio.gather(task_sensor_update, task_peripheral)

asyncio.run(main())

```

---

## Peripheral/GATT Client 温度を読み出してみる

