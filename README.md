# Busylight Driver


An unofficial driver for communicating with Busylight devices. For more details related to the device, check out the official page here: http://busylight.com/

Some of the other implemenations:

* node
   1. https://github.com/porsager/busylight
         
* python 
   1. https://github.com/mccarthyryanc/busylight    
   2. https://github.com/ericpulvino/pyBusylight


----
The project `busylight-core` is a complete and comprehensive driver implementation of the protocol specification of Busylight for [revision 2.2](docs/Busylight.API.rev.2.2.-.22052015.pdf). The purpose of this module/project is to only act as a driver for communicating with the busylight device. Most common interactions are already covered in the "high-level" api. For custom specification, you can use the `ProtocolSpec#builder`    

The implementation is structured logically into the following ---
    
## High-level Api:

* [`SpecConstants`](src/main/java/com/fyayc/essen/busylight/core/protocol/SpecConstants.java)

    Contains most common used specs. For most cases, this should be sufficient. 
    
## Low-level Api:

*   [`ProtocolSpec`](src/main/java/com/fyayc/essen/busylight/core/protocol/ProtocolSpec.java)
*   [`ProtocolStep`](src/main/java/com/fyayc/essen/busylight/core/protocol/ProtocolStep.java)
*   [`*.protocol.bytes.*`](src/main/java/com/fyayc/essen/busylight/core/protocol/bytes/)
        
## Driver
The `Driver` class implements the logic to send the raw byte buffer data to the `HidDevice`. Apart from this, the class is also responsible for maintaining a  _stable_ connection to the device. That means, it _observes_ for attachments, and detachment of the device and tries to reestablish the connection. 

Additionally, the driver is responsible for sending out this so-called **_Keep-Alive**_ command signal to the device (--to keep the previous data signal alive) periodically. This is means the scenario of attachment, followed by a detachment is handled.


## Usage 

#### Pre-defined specs 
```java
try (Driver driver = Driver.tryAndAcquire()) {
    System.out.println("****** PLAYING MUSIC *******");
    driver.send(SpecConstants.toneSpec(Tone.forTone(Tones.OPENOFFICE, 1)));
   
    System.out.println("****** PLAYING STANDARD SPECS *******");
    for (StandardSpecs spec : StandardSpecs.values()) {
      driver.send(spec);
      Thread.sleep(2000);
    }
   
    System.out.println("****** PLAYING LIGHT SPECS *******");
   
    for (Light light : Light.values()) {
      driver.send(SpecConstants.lightSpec(light));
      Thread.sleep(1000);
    }
   
    System.out.println("****** PLAYING BLINK SPECS *******");
   
    for (Light light : Light.values()) {
      driver.send(SpecConstants.blinkSpec(light));
      Thread.sleep(5000);
    }
    driver.send(StandardSpecs.OFF);
}
```
#### Custom spec
For eg, consider the following specification from the [protocol documentation](docs/Busylight.API.rev.2.2.-.22052015.pdf): 

* Step 0: Blink blue (20% intensity) 3 times (DC:0.5s on 0.5s off), turn off audio; then jump to step 1
* Step 1: Blink red (80% intensity) 5 times (DC:1.5s on 0.5s off), audio setting ignored; then jump to step 2 
* Step 2: Show green (100% intensity) steady on for 10 seconds and play ringtone #4 at volume level 3;
then jump to step 0


this can be defined as follows: 
```java
try (Driver driver = Driver.tryAndAcquire()) {
      ProtocolSpec spec =
          ProtocolSpec.builder()
              .addStep(
                  ProtocolStep.builder()
                      .light(Color.EMPTY, Color.EMPTY, Color.ofIntensity(20))
                      .lightDuration(Time.forDuration(0.5), Time.forDuration(0.5))
                      .repeat(Repeat.ofTimes(3))
                      .tone(Tone.TURN_OFF_TONE)
                      .command(Command.nextStep(1))
                      .build())
              .addStep(
                  ProtocolStep.builder()
                      .light(Color.ofIntensity(80), Color.EMPTY, Color.EMPTY)
                      .lightDuration(Time.forDuration(1.5), Time.forDuration(0.5))
                      .repeat(Repeat.ofTimes(5))
                      .tone(Tone.noSettings())
                      .command(Command.nextStep(2))
                      .build())
              .addStep(
                  ProtocolStep.builder()
                      .light(Color.EMPTY, Color.ofIntensity(80), Color.EMPTY)
                      .lightDuration(Time.forDuration(10), Time.forDuration(0))
                      .tone(Tone.forTone(4,3))
                      .command(Command.nextStep(0))
                      .build())
              .build();
      driver.send(spec);
    }
```
---
Unless, specified otherwise, all parts code in all the branches in this repository is  subjected to [GNU AGPLV3 License](https://github.com/tckb/busylight_driver/blob/master/LICENSE)
