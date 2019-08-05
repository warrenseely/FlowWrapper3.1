# FlowWrapper3.1
Latest Version of the flow wrapper control code implemented 4/25/2019
/*UPDATE April 25, 2019: 
   * Added secondary layer of control for jaws to help alignment. This is based on comparing time snapshots from the jaws and register mark to verify encoder position of jaws, similar to the secondary layer utilized for the feeder chain in V3.1 
   * Also cleared encoder counters when stop button is pressed to help prevent surging upon startup after a jammed feeder chain
   * 
   * Implemented April 8, 2019. 
   * V3.1 introduces encoder control of jaws. This is coupled with a second jaw to enable continuous running.
   * Also includes a 400ms debounce for the register sensor, jaw position sensor and the feeder finger sensor
   * NOTE that this limits max speed to around 150 pkg/min
   * Also firms out the second layer of control using time snapshots to verify the encoder position of jaws and feeder chain relative to the paper register mark.
   *        This works by computing the time difference between when the register mark and the feeder finger/jaw home sensor are seen.
   *        The encoder values for jaws and feeder are then manipulated to convince the PID loops to adjust as necessary.
   * NOTE pin assignment changes from version 3.0: 
   *        PaperEncoder_PIN 19 --> 0
   *        FeederFinger_PIN 8 --> 19
   *  
   *  
   *  V3.0 introduces control by an Arduino Mega rather than a Nano for more pins and controls
   * Also introduces display screen controlled by and Arduino Nano for better data viewing
   * NOTE THAT PINS 20 && 21 ARE RESERVED FOR DISPLAY SCREEN i2C!
   * NOTE also that PJ0 and PJ1 do not actually support PCINT despite what the manual says
   * NOTE that jawPWM and feederPWM(pins 11 and 12) are 10 bit control, so 0-1023 INVERTED
   *           paperPWM(pin 13) is 8 bit control, so 0-255 INVERTED
   *     *****The reason these are inverted PWM is due to the way the control circuit is designed; Thus, 10v to vfd(full speed) is 0 PWM and 0v to vfd is 255/1023 PWM******
   * NOTE no interrupts on pin 3 for some reason
   * 
   * TO DO: convert speed input to pkg/min(estimated) to aid setting
   *        maybe dynamic(on the fly) speed changes
   *        communication upgrades(non blocking when waiting for ack)
   *        
   * 
   * V2.2 fixed the feeder chain surging issues is previous versions.
   * Also works in tandem with another arduino to perform no gap no crimp to avoid crushing mints.
   * 
   * V2.1 gives priming of the feeder chain on startup with mints prior to running
   * Also introduces LCD display communication on the tx channel of USB
   * Works simultaneously with the USB plugged in, although LCD commands show as garbage
   * Note that overloaded function writeLCD() can only print those types of data it is overloaded for
   * Also introduces a 1.5 minute startup delay on power up to allow the VFDs to unscramble their brains
   * Also introduces checks to ensure jaws make complete cycle during startup to avoid weird pwm bugs resulting from less than full cycle
   * Also disables pin change interrupts when stop button pressed, reenables when start button pressed UNTESTED
   * Also introduces a timing comparison between the paper register and feeder fingers to adjust for accumulated encoder error
   *
   * V2.0 introduced PID control of the feeder chain to match the paper speed. 
  Also introduced Timer1 set as 16bit to enable 10bit pwm on pins 9 and 10(jaw and feeder pwm) for greater resolution(1024 steps vs 255)
  Also utilizes encoders on paper drive and feeder drive to match speeds correctly utilizing the PID control.
  This version introduces a timed-loop control for consistent response time for both button commands and run operation.
  includes live printout of speed errors, and current PID scalars along with ability to adjust during run operation WITHOUT SAVING
  Also splits the jaw and paper register interrupts into separate pin change ISRs.
  Note that feeder speed is adjusted by PID, but jaw speed is hard coded in this version. 
  Jaw speed is still stop/go control.
  Note that the PID runs off of incrementing pulses on the encoders; 
  Encoder counts utilize LONG int, which should give 1,491.3 hours of continouos run time(without pressing stop) prior to overrun*/

//Encoder wiring:
//Green: A-phase---not used at this time
//White: B-phase
//Red: +5-24v
//Black: Ground

/*Paper speed is MASTER
  Jaws run based on paper speed
  feeder chain runs based off paper speed*/


/*******************IMPORTANT!!!!!********************
*************Pkg dimensions are in METRIC*************
*******************************************************/
// -- > https://github.com/JChristensen/Button/blob/master/Button.cpp
#include <Button.h> //include this library for convenient reading of the buttons
#include <Wire.h> //communication to the display screen arduino
