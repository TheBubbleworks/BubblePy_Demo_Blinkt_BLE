# BubblePy Blinkt BLE + Web Bluetooth Demo

This is a BubblePy demo of controlling Pimoroni Blinkt lights via BluetoothLE.

The User Interface runs in a browser and uses the [Web Bluetooth JavaScript API](https://github.com/WebBluetoothCG/web-bluetooth#web-bluetooth) for discovering and connecting to Bluetooth LE devices from within a browser, a native app or plugin on Desktop or Mobile is not required.

The Python code demonstrates BubblePy's MicroPython versions of [gpiozero](https://github.com/RPi-Distro/python-gpiozero) and [bluezero](https://github.com/ukBaz/python-bluezero). This code runs on multiple target Bluetooth LE enabled devices (more details below).


## User Interface

Open the Web Bluetooth demo in a using this [link](https://thebubbleworks.github.io/BubblePy_Demo_Blinkt_BLE/).

*Note: Please check your platform and browser are supported [here](https://github.com/WebBluetoothCG/web-bluetooth/blob/gh-pages/implementation-status.md).*


## Python Code

The following Python code runs *almost* unmodified on [BubblePy](https://thebubbleworks.com/bubblepy/) on these devices:

- Arduino Primo
- BBC micro:bit
- Raspberry Pi
- Other Nordic Semiconductor nRF51 and nRF52 devices, e.g. RuuviTag, Red Bear LAbs Nano, Adafruit Feather

**The DAT and CLK pins may need changing for your hardware platform.*


```python
from gpiozero import OutputDevice
from bluezero import peripheral

DAT = 23
CLK = 24
NUM_PIXELS = 8
BRIGHTNESS = 7

SERVICE_UUID =  '0000fff0-0000-1000-8000-00805f9b34fb'
CHAR_UUID =    '0000fff3-0000-1000-8000-00805f9b34fb'
#SERVICE_UUID = '6e400001-b5a3-f393-e0a9-e50e24dcca9e'
#CHAR_UUID = '6e400002-b5a3-f393-e0a9-e50e24dcca9e'


# Commands, received via BLE
CMD_CLEAR_PIXELS=0x0601
CMD_SET_PIXEL=0x0702


dat = OutputDevice(DAT)
clk = OutputDevice(CLK)

pixels = [[0,0,0,BRIGHTNESS]] * 8


def set_brightness(brightness):
    for x in range(NUM_PIXELS):
        pixels[x][3] = int(31 * brightness) & 0b11111

def clear():
    for x in range(NUM_PIXELS):
        pixels[x][0:3] = [0,0,0]

def write_byte(byte):
    for x in range(8):
        bit = (byte & (1 << (7-x))) > 0
        if bit:
            dat.on()
        else:
            dat.off()
        clk.on()
        clk.off()

def show():
    for x in range(4):
        write_byte(0)

    for pixel in pixels:
        r, g, b, brightness = pixel
        write_byte(0b11100000 | brightness)
        write_byte(b)
        write_byte(g)
        write_byte(r)

    write_byte(0xff)

def set_pixel(x, r, g, b, brightness=None):
    if brightness is None:
        brightness = pixels[x][3]
    else:
        brightness = int(31 * brightness) & 0b11111
    pixels[x] = [int(r) & 0xff,int(g) & 0xff,int(b) & 0xff,brightness]


def set_all(r,g,b):
    for x in range(8):
        set_pixel(x,r,g,b)


def ble_write_callback(bytes):
    if len(bytes)>2:
        cmd = (bytes[0]<<8) + (bytes[1] & 0xff)

        if cmd == CMD_SET_PIXEL:
            if len(bytes) >=7 :
                set_pixel(bytes[3]-1, bytes[4], bytes[5], bytes[6] )
        elif cmd == CMD_CLEAR_PIXELS:
            if len(bytes) >=5:
                set_all(bytes[2], bytes[3], bytes[4] )
    show()

service = peripheral.Service(SERVICE_UUID, True)
char = peripheral.Characteristic(CHAR_UUID, ['write'], 0)

char.add_write_event(ble_write_callback)
service.add_characteristic(char)

peripheral.add_service(service)
peripheral.start()
```