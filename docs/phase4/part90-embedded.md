# Part 90: Embedded Systems ด้วย TinyGo

## เป้าหมายของบทเรียน
- TinyGo สำหรับ microcontrollers
- Raspberry Pi GPIO programming
- Arduino/AVR development
- I2C, SPI, UART protocols
- Sensor reading และ control

---

## 1. TinyGo Overview

```go
// tinygo_intro.go - TinyGo introduction
package main

import (
    "fmt"
)

/*
TinyGo:
- Go compiler สำหรับ embedded/IoT devices
- ใช้ LLVM backend
- รองรับ microcontrollers: Arduino, STM32, Nordic nRF, ESP32, RP2040
- รองรับ SBCs: Raspberry Pi, BeagleBone
- รองรับ WASM

Target devices:
- Arduino Uno/Mega/Nano (ATmega328P/2560)
- Arduino Zero (SAMD21)
- Nordic nRF52840 Dongle
- Raspberry Pi Pico (RP2040)
- ESP32/ESP8266
- STM32 (Blue Pill, etc.)

Install TinyGo:
  # macOS
  brew install tinygo
  
  # Linux
  wget https://github.com/tinygo-org/tinygo/releases/download/v0.31.0/tinygo0.31.0.linux-amd64.tar.gz
  tar -C /usr/local -xzf tinygo0.31.0.linux-amd64.tar.gz
  
  # Windows
  winget install TinyGo

Flash to device:
  tinygo flash -target arduino ./
  tinygo flash -target pico ./
  tinygo flash -target esp32-coreboard-v2 ./
*/

func main() {
    fmt.Println("TinyGo Embedded Systems")
    
    targets := []struct {
        name  string
        chip  string
        flash string
        ram   string
    }{
        {"Arduino Uno", "ATmega328P", "32KB", "2KB"},
        {"Arduino Mega", "ATmega2560", "256KB", "8KB"},
        {"Raspberry Pi Pico", "RP2040", "2MB", "264KB"},
        {"Arduino Zero", "SAMD21", "256KB", "32KB"},
        {"Nordic nRF52840", "nRF52840", "1MB", "256KB"},
        {"ESP32", "Xtensa LX6", "4MB", "520KB"},
    }
    
    fmt.Printf("\n%-20s %-15s %-8s %-8s\n", "Device", "Chip", "Flash", "RAM")
    fmt.Println(string(make([]byte, 55)))
    for _, t := range targets {
        fmt.Printf("%-20s %-15s %-8s %-8s\n", t.name, t.chip, t.flash, t.ram)
    }
}
```

---

## 2. GPIO Control

```go
//go:build tinygo && (arduino || pico)

// gpio_led.go - LED control บน Arduino/Pico
package main

import (
    "machine"
    "time"
)

// BlinkLED กระพริบ LED
func BlinkLED() {
    led := machine.LED
    led.Configure(machine.PinConfig{Mode: machine.PinOutput})
    
    for {
        led.High()
        time.Sleep(500 * time.Millisecond)
        
        led.Low()
        time.Sleep(500 * time.Millisecond)
    }
}

// ButtonLED ควบคุม LED ด้วยปุ่ม
func ButtonLED() {
    // LED output
    led := machine.LED
    led.Configure(machine.PinConfig{Mode: machine.PinOutput})
    
    // Button input with pull-up
    btn := machine.D2
    btn.Configure(machine.PinConfig{Mode: machine.PinInputPullup})
    
    for {
        if !btn.Get() { // Active low (pull-up)
            led.High()
        } else {
            led.Low()
        }
        time.Sleep(10 * time.Millisecond)
    }
}

// PWMFade fade LED ด้วย PWM
func PWMFade() {
    pwm := machine.PWM0
    pwm.Configure(machine.PWMConfig{
        Period: 1e9 / 1000, // 1kHz
    })
    
    ch, _ := pwm.Channel(machine.LED)
    
    for {
        // Fade in
        for i := 0; i <= 255; i++ {
            pwm.Set(ch, uint32(i*pwm.Top()/255))
            time.Sleep(5 * time.Millisecond)
        }
        
        // Fade out
        for i := 255; i >= 0; i-- {
            pwm.Set(ch, uint32(i*pwm.Top()/255))
            time.Sleep(5 * time.Millisecond)
        }
    }
}

func main() {
    BlinkLED()
}
```

---

## 3. I2C Protocol

```go
//go:build tinygo

// i2c_sensor.go - I2C sensor reading (BMP280 temperature/pressure)
package main

import (
    "fmt"
    "machine"
    "time"
)

const (
    BMP280_ADDR = 0x76 // I2C address

    // Registers
    REG_CHIP_ID     = 0xD0
    REG_RESET       = 0xE0
    REG_STATUS      = 0xF3
    REG_CTRL_MEAS   = 0xF4
    REG_CONFIG      = 0xF5
    REG_PRESS_MSB   = 0xF7
    REG_TEMP_MSB    = 0xFA
    REG_CALIB_START = 0x88
)

// BMP280 temperature/pressure sensor
type BMP280 struct {
    bus  machine.I2C
    addr uint16
    
    // Calibration data
    digT1 uint16
    digT2 int16
    digT3 int16
    digP1 uint16
    digP2 int16
    digP3 int16
    digP4 int16
    digP5 int16
    digP6 int16
    digP7 int16
    digP8 int16
    digP9 int16
}

// NewBMP280 สร้าง sensor instance
func NewBMP280(bus machine.I2C) *BMP280 {
    return &BMP280{
        bus:  bus,
        addr: BMP280_ADDR,
    }
}

// Init เริ่มต้น sensor
func (b *BMP280) Init() error {
    // ตรวจสอบ chip ID
    chipID, err := b.readReg(REG_CHIP_ID)
    if err != nil {
        return fmt.Errorf("read chip ID: %w", err)
    }
    
    if chipID != 0x60 {
        return fmt.Errorf("unexpected chip ID: 0x%02X", chipID)
    }
    
    // อ่าน calibration data
    if err := b.readCalibration(); err != nil {
        return fmt.Errorf("read calibration: %w", err)
    }
    
    // ตั้งค่า normal mode, oversampling x1
    b.writeReg(REG_CTRL_MEAS, 0x27)
    b.writeReg(REG_CONFIG, 0x00)
    
    return nil
}

// readReg อ่าน register 1 byte
func (b *BMP280) readReg(reg byte) (byte, error) {
    buf := make([]byte, 1)
    err := b.bus.Tx(b.addr, []byte{reg}, buf)
    return buf[0], err
}

// writeReg เขียน register
func (b *BMP280) writeReg(reg, value byte) {
    b.bus.Tx(b.addr, []byte{reg, value}, nil)
}

// readCalibration อ่าน calibration coefficients
func (b *BMP280) readCalibration() error {
    calib := make([]byte, 24)
    err := b.bus.Tx(b.addr, []byte{REG_CALIB_START}, calib)
    if err != nil {
        return err
    }
    
    b.digT1 = uint16(calib[1])<<8 | uint16(calib[0])
    b.digT2 = int16(calib[3])<<8 | int16(calib[2])
    b.digT3 = int16(calib[5])<<8 | int16(calib[4])
    b.digP1 = uint16(calib[7])<<8 | uint16(calib[6])
    b.digP2 = int16(calib[9])<<8 | int16(calib[8])
    // ... อ่านต่อ
    
    return nil
}

// ReadTemperature อ่านอุณหภูมิ (องศาเซลเซียส)
func (b *BMP280) ReadTemperature() (float32, error) {
    raw, err := b.readRaw(REG_TEMP_MSB)
    if err != nil {
        return 0, err
    }
    
    // Compensation formula จาก datasheet
    var1 := ((int32(raw) >> 3) - (int32(b.digT1) << 1)) * int32(b.digT2) >> 11
    var2 := (((int32(raw) >> 4) - int32(b.digT1)) * ((int32(raw) >> 4) - int32(b.digT1)) >> 12) * int32(b.digT3) >> 14
    tFine := var1 + var2
    
    temp := (tFine*5 + 128) >> 8
    return float32(temp) / 100.0, nil
}

// readRaw อ่าน 20-bit raw value
func (b *BMP280) readRaw(reg byte) (int32, error) {
    buf := make([]byte, 3)
    err := b.bus.Tx(b.addr, []byte{reg}, buf)
    if err != nil {
        return 0, err
    }
    
    raw := int32(buf[0])<<12 | int32(buf[1])<<4 | int32(buf[2])>>4
    return raw, nil
}

func main() {
    // Init I2C
    i2c := machine.I2C0
    i2c.Configure(machine.I2CConfig{
        Frequency: 400000, // 400kHz
        SDA:       machine.SDA,
        SCL:       machine.SCL,
    })
    
    // Init sensor
    sensor := NewBMP280(i2c)
    if err := sensor.Init(); err != nil {
        fmt.Printf("Sensor init error: %v\n", err)
        return
    }
    
    fmt.Println("BMP280 initialized successfully")
    
    // Read loop
    for {
        temp, err := sensor.ReadTemperature()
        if err != nil {
            fmt.Printf("Read error: %v\n", err)
        } else {
            fmt.Printf("Temperature: %.2f°C\n", temp)
        }
        
        time.Sleep(2 * time.Second)
    }
}
```

---

## 4. UART Communication

```go
//go:build tinygo

// uart_comm.go - UART serial communication
package main

import (
    "machine"
    "strconv"
    "time"
)

// UARTProtocol จัดการ UART communication
type UARTProtocol struct {
    uart    machine.UART
    buf     []byte
    pos     int
}

// NewUARTProtocol สร้าง instance
func NewUARTProtocol(u machine.UART) *UARTProtocol {
    return &UARTProtocol{
        uart: u,
        buf:  make([]byte, 256),
    }
}

// Send ส่งข้อมูล
func (p *UARTProtocol) Send(data []byte) {
    p.uart.Write(data)
}

// SendLine ส่ง line
func (p *UARTProtocol) SendLine(msg string) {
    p.uart.Write([]byte(msg + "\r\n"))
}

// ReadLine อ่าน line
func (p *UARTProtocol) ReadLine() (string, bool) {
    for p.uart.Buffered() > 0 {
        b, _ := p.uart.ReadByte()
        
        if b == '\n' {
            line := string(p.buf[:p.pos])
            p.pos = 0
            // ลบ \r ถ้ามี
            if len(line) > 0 && line[len(line)-1] == '\r' {
                line = line[:len(line)-1]
            }
            return line, true
        }
        
        if p.pos < len(p.buf) {
            p.buf[p.pos] = b
            p.pos++
        }
    }
    
    return "", false
}

// ATCommand ส่ง AT command และรอ response
func (p *UARTProtocol) ATCommand(cmd string, timeout time.Duration) string {
    p.SendLine("AT+" + cmd)
    
    deadline := time.Now().Add(timeout)
    response := ""
    
    for time.Now().Before(deadline) {
        if line, ok := p.ReadLine(); ok {
            response += line + "\n"
            if line == "OK" || line == "ERROR" {
                break
            }
        }
        time.Sleep(time.Millisecond)
    }
    
    return response
}

// ESP8266WiFi จำลอง ESP8266 WiFi module via UART
type ESP8266 struct {
    proto *UARTProtocol
}

func NewESP8266(u machine.UART) *ESP8266 {
    return &ESP8266{
        proto: NewUARTProtocol(u),
    }
}

func (e *ESP8266) Connect(ssid, password string) error {
    // Set mode: station
    resp := e.proto.ATCommand(`CWMODE=1`, 2*time.Second)
    if resp == "" {
        return nil // Mock success
    }
    
    // Connect to AP
    cmd := `CWJAP="` + ssid + `","` + password + `"`
    e.proto.ATCommand(cmd, 10*time.Second)
    
    return nil
}

func (e *ESP8266) GetIP() string {
    e.proto.ATCommand("CIFSR", 2*time.Second)
    return "192.168.1.100" // Mock
}

func main() {
    // Init UART
    uart := machine.UART0
    uart.Configure(machine.UARTConfig{
        BaudRate: 115200,
        TX:       machine.UART0_TX,
        RX:       machine.UART0_RX,
    })
    
    proto := NewUARTProtocol(uart)
    
    // Send hello
    proto.SendLine("Hello from TinyGo!")
    
    // Echo loop
    counter := 0
    for {
        if line, ok := proto.ReadLine(); ok {
            proto.SendLine("Echo: " + line + " (#" + strconv.Itoa(counter) + ")")
            counter++
        }
        time.Sleep(10 * time.Millisecond)
    }
}
```

---

## 5. Simulation สำหรับ Testing บน Desktop

```go
// embedded_sim.go - จำลอง embedded environment บน desktop
//go:build !tinygo

package main

import (
    "fmt"
    "math/rand"
    "time"
)

// SimPin จำลอง GPIO pin
type SimPin struct {
    name  string
    state bool
    mode  string
}

func (p *SimPin) Configure(mode string) {
    p.mode = mode
    fmt.Printf("Pin %s configured as %s\n", p.name, mode)
}

func (p *SimPin) High() {
    p.state = true
    fmt.Printf("Pin %s: HIGH\n", p.name)
}

func (p *SimPin) Low() {
    p.state = false
    fmt.Printf("Pin %s: LOW\n", p.name)
}

func (p *SimPin) Get() bool {
    return p.state
}

func (p *SimPin) Toggle() {
    if p.state {
        p.Low()
    } else {
        p.High()
    }
}

// SimI2C จำลอง I2C bus
type SimI2C struct {
    devices map[uint8]SimI2CDevice
}

// SimI2CDevice จำลอง I2C device
type SimI2CDevice interface {
    ReadRegister(reg byte) byte
    WriteRegister(reg, value byte)
}

// SimBMP280 จำลอง BMP280 sensor
type SimBMP280 struct {
    baseTemp float32
}

func NewSimBMP280(baseTemp float32) *SimBMP280 {
    return &SimBMP280{baseTemp: baseTemp}
}

func (s *SimBMP280) ReadRegister(reg byte) byte {
    switch reg {
    case 0xD0: // Chip ID
        return 0x60
    default:
        return 0
    }
}

func (s *SimBMP280) WriteRegister(reg, value byte) {}

func (s *SimBMP280) ReadTemperature() float32 {
    noise := (rand.Float32() - 0.5) * 0.5
    return s.baseTemp + noise
}

func (s *SimBMP280) ReadPressure() float32 {
    return 1013.25 + (rand.Float32()-0.5)*5
}

// EmbeddedApp จำลอง embedded application
type EmbeddedApp struct {
    led    *SimPin
    btn    *SimPin
    sensor *SimBMP280
}

func NewEmbeddedApp() *EmbeddedApp {
    return &EmbeddedApp{
        led:    &SimPin{name: "LED"},
        btn:    &SimPin{name: "BUTTON"},
        sensor: NewSimBMP280(25.0),
    }
}

func (app *EmbeddedApp) Setup() {
    fmt.Println("=== Embedded App Setup ===")
    app.led.Configure("output")
    app.btn.Configure("input_pullup")
    fmt.Println("Sensors initialized")
    fmt.Println()
}

func (app *EmbeddedApp) Loop(iterations int) {
    fmt.Printf("=== Running %d iterations ===\n", iterations)
    
    for i := 0; i < iterations; i++ {
        // Read sensor
        temp := app.sensor.ReadTemperature()
        pressure := app.sensor.ReadPressure()
        
        fmt.Printf("[%d] Temp: %.2f°C  Pressure: %.2f hPa\n",
            i+1, temp, pressure)
        
        // Blink based on temperature
        if temp > 26.0 {
            app.led.High()
        } else {
            app.led.Low()
        }
        
        time.Sleep(500 * time.Millisecond)
    }
}

func main() {
    rand.Seed(time.Now().UnixNano())
    
    app := NewEmbeddedApp()
    app.Setup()
    app.Loop(6)
    
    fmt.Println("\n=== Flash to Device ===")
    fmt.Println("tinygo flash -target arduino ./")
    fmt.Println("tinygo flash -target pico ./")
    fmt.Println("tinygo flash -target esp32-coreboard-v2 ./")
    fmt.Println("\n=== Monitor Serial ===")
    fmt.Println("tinygo monitor -target arduino")
}
```

---

## สรุป

บทนี้ครอบคลุม Embedded Systems ด้วย TinyGo:

1. **TinyGo Overview** - targets, installation, capabilities
2. **GPIO** - digital I/O, LED, buttons, PWM
3. **I2C** - sensor communication (BMP280)
4. **UART** - serial communication, AT commands
5. **Desktop Simulation** - testing without hardware

### Key Takeaways

- TinyGo ทำให้ใช้ Go บน microcontrollers ได้
- `machine` package abstracts hardware peripherals
- ขนาดไฟล์เล็กมาก (เหมาะกับ 2KB RAM devices)
- Test บน desktop ก่อน flash ไปที่ device จริง
- I2C/SPI เป็น protocol มาตรฐานสำหรับ sensors
