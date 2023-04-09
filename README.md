
These are boards for C128D/DCR internal WiFi modems based on the ACIA 6551 and Wemos D1 ESP8266 modules flashed with Zimodem firmware.

This work is heavily base on dabonetn's work from [Link232-Wifi](https://github.com/dabonetn/Link232-Wifi) project. Please go there for additional documentation.

# Installation

For both projects you need to decide on the interrupt source and address space. I recommend using `/NMI` as the interrupt and `$D700` as the base address.
This way, you can readily use most popular software for both C128 (DesTerm) and C64 (CCGMS), and the mod doesn't interfere with cartridges.

## Raise CIA

For both boards one of the CIA chips is raised to the daughterboard.

## Interrupt line

Both boards connect ACIA interrupt pin to local CIA chip but if it's not the right one (i.e. you want `/NMI` but the board is over CIA#1 which is connected to `/IRQ`) you need to run a wire to a suitable singal source from the mainboard.

## Base address

A signal for ACIA I/O address access in $D700-$D7FF area is readily available from U3 pin 12 that is otherwise unconnected.

## DotClock

Both boards require DotClock signal from the Expansion port pin 6.

# Swift-L (C128D)

This board is meant primarly for C128D and it is supposed to be plugged in place of CIA#2. It is a clone of [Link232 Swift-L](https://github.com/dabonetn/Link232-Wifi) project converted to Kicad6, with Burst connector added.

The `/NMI` signal is already available from the mainboard CIA#2 socket, so you only need to connect ACIA I/O signal and DotClock.

This board also features a 10-pin socket for a flat cable for burst connection to 1541 drives. The pinout is compatible with [DolphinDos3](https://e4aws.silverdr.com/projects/dolphindos3/) with `/FLAG` signal on pin 1, then 8 data lines and `/PC2` signal on pin 10. 

## C128DCR notes

There is not enough clearance under power supply on C128DCR to use that board with CIA#2, but this should fit in place of CIA#1.

You need to route `/NMI` interrupt line to pin 26 of ACIA 6551 somewhere from the board - the Expansion port pin D is the closet spot.

The burst connector would work correctly, but no software uses CIA#1 for that purpose.

## Swift-CIA3 (C128DCR)

This board is meant to be used in C128DCR over CIA#1. It has several features that I needed and you may find optional. If all you need is internal ACIA modem you should better use Swift-L variant.

You need to connect these external control signals to connector J1:

|pin|  |
|---|--|
|1|I/O signal for ACIA, for page $D700 connect to U3 pin 12|
|2|DotClock from Expansion port pin 6|
|3|Interrupt for ACIA, connect `/NMI` from Expansion port pin D|
|4|I/O signal for CIA#3 (optional)

In my case both ACIA I/O and CIA#3 I/O signals come from [U3 address decoder replaced by a GAL](https://github.com/ytmytm/c128-u3-replacement), so CIA#1 occupies addresses $DC00-$DC20 and CIA#3 appears in $DC20-$DC40

Connector J2 has a simple serial bus with GND/VCC/SP/CNT signals from CIA#3.

Connector J4 exposes CIA#3 port B so that it can be used to switch Kernal or Internal Function ROMs.

A jumper controls whether ACIA is connected to `/IRQ` (from CIA#1) or to `/NMI` (from J1 pin 3).

Another jumper selects 50/60Hz signal for TOD clock coming from ATTiny.

### CIA#3

CIA#3 is connected to ATA-IDE 40-pin connector for harddrives for [my CIAIDE project](https://github.com/ytmytm/c64-ciaide).
Port B lines are also connected to a pin header because they could be used for something like an internal ROM switcher.

### ATTiny85

There is a place for ATTiny85 to generate TOD clock signal.

In one of my C128DCRs I replaced power supply by a Mean Well RD35A power supply with 12V(1A)/5V(4A) rails. This runs much cooler than original power supply, but doesn't generate 9VAC. I don't mind the tape port as I don't use it anyway, but TOD clock is useful for GEOS.

Note that without 9VAC you need to replace CR8 by a wire (to short it to GND so that input to U16 pin 11 is not floating) and completely remove R7 and C80 elements.

ATTiny runs this simple Arduino code to generate 50/60Hz waveform on TOD clock line, depending on jumper being open or closed:

```

// IDE: ATtiny25/45/85, internal 1MHz
// 1MHz is the default speed, no need to switch fuses to 8MHz
// compile to HEX (Sketch->Export compiled binary) and burn using TL866-II

// pinout
//   /RESET 1  U  8 5V
//   A3/PB3 2     7 A1/2/PB2
//   A2/PB4 3     6 PB1 PWM
//      GND 4     5 PB0 PWM -- output PWM 50Hz

const uint8_t PWM_pin = 0; // PB0, pin 5
const uint8_t mode_pin = 1; // PB1, pin 6 - jumper to GND for NTSC mode, keep floating (internal pullup) for PAL
uint8_t half_duty = 10;    // 10ms for PAL, 8ms for NTSC = 20ms/16ms

void setup() {
  pinMode(PWM_pin, OUTPUT);
  digitalWrite(PWM_pin, LOW);
  pinMode(mode_pin, INPUT_PULLUP);
  delay(200);
  if (digitalRead(mode_pin) == LOW) { // is it tied to GND?
    half_duty = 8;  // yes - switch to 8ms half duty cycle for 16ms ~ 62.5Hz frequency
  }
}

void loop() {
  // could be done using PWM and clock prescalers, but that is good enough for me
  while (true) {
    delay(half_duty); // 10ms - half of 20ms duty cycle
    digitalWrite(PWM_pin, HIGH);
    delay(half_duty); // 10ms - half of 20ms duty cycle
    digitalWrite(PWM_pin, LOW);
  }
}

```

# Zimodem - Wemos D1 firmware

Flash Wemos D1 with [Zimodem](https://github.com/bozimmerman/Zimodem) software.

*Attention*: Check if this notice still exists in the README. If it does - make sure to downgrade Generic ESP8266 Library to version 2.7.4 in your Arduino IDE.

```
2.7.4 Arduino ESP8266 libraries using the Generic ESP8266 Module.  Versions newer than 2.7.4 may not work.
```

*Attention*: Search for these defines and make these changes on the top of zimodem.ino before compiling:

1. Comment out this line to make RS232 lines non-inverted by default:
```
//# define RS232_INVERTED 1
```

2. Set default baud rate to 19200 (sensible choice for C64) or 38400 (maximum for ACIA)
```
#define DEFAULT_BAUD_RATE 19200
```

# C64/128 software

With these changes and ACIA setup to use `/NMI` at $D700 base address both latest version of CCGMS and DesTerm 128 will work fine.

