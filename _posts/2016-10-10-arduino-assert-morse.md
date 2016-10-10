---
layout: post
title:  "Arduino, transmit an assertion failure message through Morse code"
date:   2016-10-10 10:00:00 +0100
comments: true
---

Debugging on Arduino is not always easy. On a desktop application, assertions are a good tool to help us debug programs by checking if some invariants hold true, and if not to abort the program with an error. On Arduino we are more in the dark. By default if an assertion fails the controller disables interruptions and go into an infinite empty loop. Not really practical.

One way to overcome this would be to connect the Arduino to a computer, write the error to the serial port and read the port from the computer. We could also write the error into the [EEPROM](https://www.arduino.cc/en/Reference/EEPROM) and read it again when we plug the Arduino to a computer. Or if we have an XBee, Bluetooth, Wi-Fi or any other wireless module, we could transmit the error this way.

But then I saw I had a little buzzer left and I thought, wouldn't it be more fun to use some Morse code to signal the error?

So here we go, an assert function which takes a condition and a message in parameters and play the message in Morse code through the buzzer if the condition is false before aborting.

# Code

First we will need to define the assert macro which takes an additional message in parameter. Let's call it assertp.

```c++
#ifdef NDEBUG
#define assertp(expr, message) ((void)0)
#else
#define assertp(expr, message) \
((expr) ? (void) 0 : assertp_fail(#expr, __FILE__, __LINE__, message))
#endif
```

Then we need to define the assertp_fail function. The function will send the error through the serial port before playing the message in Morse.

```c++
inline void assertp_fail(const char* expression, const char* file, 
                         int line, const char* message) 
{
    Serial.println(expression);
    Serial.println(file);
    Serial.println(line);
    Serial.println(message);
    
    MorseSpeaker morseSpeaker(Pin::SPEAKER_PIN);
    morseSpeaker.playSentence(message);
    
    abort();
}
```

Now we just need the MorseSpeaker class which will play the message in Morse. The code for this class, which is quite simple and will not be detailed here, can be found on [GitHub](https://github.com/Tessil/arduino-robot/blob/master/src/MorseSpeaker.cpp).

```c++
assertp(1 + 1 == 2, "We are in trouble");
```


The only thing left to do for you now is to learn the Morse alphabet.
