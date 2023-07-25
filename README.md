[![PlatformIO Registry](https://badges.registry.platformio.org/packages/yeganemehr/library/arduino-sim800.svg)](https://registry.platformio.org/libraries/yeganemehr/arduino-sim800)
[![Build Examples](https://github.com/yeganemehr/arduino-sim800/actions/workflows/build-examples.yml/badge.svg)](https://github.com/yeganemehr/arduino-sim800/actions/workflows/build-examples.yml)

# General Information
I tried to build a fluent API for using Sim800 and keep developers away from working with AT commands.  
This library built on top of [yeganemehr/arduino-at](https://github.com/yeganemehr/arduino-at) which make it completely async.   
It's tested against Sim800c module & ESP8266 microprocessor but technically it can works with other boards and modules too.

# Quickstart
Lets send an SMS:

```c++
#include <Arduino.h>
#include <SoftwareSerial.h>
#include <ATConnection.hpp>
#include <Sim800.hpp>

SoftwareSerial swSerial(D3, D4); // RX & TX pins
ATConnection at(&swSerial);
Sim800 sim800(&at);

void setup()
{
    Serial.begin(9600);
    swSerial.begin(9600);
    Serial.println("Initialize Sim800 connection");
    sim800.init()
        ->onSuccess([]() {
            Serial.println("Initialize Sim800 connection [done]");
            Serial.println("Put sim800 into sms text mode");
            sim800.putInTextMode()
                ->onSuccess([]() {
                    Serial.println("Put sim800 into sms text mode [done]");

                    Serial.println("Sending text");
                    sim800.sendAsciiSMS("+100000000", "Hi")
                        ->onSuccess([](const uint32_t &id) {
                            Serial.printf("Sending text [ID: %u]\n", id);
                        })
                        ->onFail([](const std::exception &reason) {
                            Serial.println("Sending text [failed]");
                        })
                        ->freeOnFinish();
                })
                ->onFail([](const std::exception &reason) {
                    Serial.println("Put sim800 into sms text mode [failed]");
                })
                ->freeOnFinish();

        })
        ->onFail([](const std::exception &reason) {
            Serial.println("Initialize Sim800 connection [failed]");
        })
        ->freeOnFinish();
}

void loop()
{
    at.communicate();
}
```
You can build and upload this code to your board quickly with [Ready-To-Use Example](examples/send-sms)

## Methods & Helpers

**Promise\<void\> \*Sim800::init()**:  
This method is a helper to run these methods in sequence and keeps thing simple for you:
 1. Sim800::checkConnection()
 2. Sim800::disableEcho()
 3. Sim800::activeSMSNotification()

**Promise\<void\> \*Sim800::checkConnection()**:  
Sends a "AT" command on serial and wait for modem's response.

**Promise\<void\> \*Sim800::disableEcho()**:  
Sends "ATE0" command on serial and wait for modem's response.

**Promise\<void\> \*Sim800::activeSMSNotification()**:  
Active unsolicited messages from modem in event of receiving new SMS.
These notifications are accessible via `ATConnection::on(ATNotificationEvent::type, cb)`

**Promise\<void\> \*Sim800::deactiveSMSNotification()**:  
Disable unsolicited messages from modem in event of receiving new SMS.

**Promise\<void\> \*Sim800::deleteAllSMSes(DeleteSMSFilter filter = DELETE_ALL)**:  
Delete SMSes from modem's memory.
Possible filters are:
 - DeleteSMSFilter::DELETE_READ
 - DeleteSMSFilter::DELETE_UNREAD
 - DeleteSMSFilter::DELETE_SENT
 - DeleteSMSFilter::DELETE_UNSENT
 - DeleteSMSFilter::DELETE_INBOX
 - DeleteSMSFilter::DELETE_ALL (default)

**Promise\<void\> \*Sim800::deleteSMS(uint8_t index)**:  
Delete an single SMS from modem's memory.

**Promise\<void\> \*Sim800::setCurrentSMSMode(SMSMode smsMode)**:  
Put modem into selected mode.
For sending plain ascii text you should put it into `SMSMode::TEXT_MODE` mode and for sending PDUs you must use `SMSMode::PDU_MODE`.

**SMSMode Sim800::getCurrentSMSMode()**:  
If you already put the modem into a mode, this method returns current mode otherwise `SMSMode::UNKNOWN`.


**Promise\<void\> \*Sim800::putInTextMode()**:  
Same as `Sim800::setCurrentSMSMode(SMSMode::TEXT_MODE)`.

**Promise\<void\> \*Sim800::putInPDUMode()**:  
Same as `Sim800::setCurrentSMSMode(SMSMode::PDU_MODE)`.

**Promise\<void\> \*Sim800::setBaudRate(uint32_t rate)**:  
Sends `AT+IPR` command and fix new baud rate with your modem.
If you pass 0 as rate to it auto-bauding will activate.

**Promise\<int8_t\> \*Sim800::getSignalLevel()**:  
Sends `AT+CSQ` command to the modem and extracts current RSSI value.
Valid values must be in -115dBm to -52dBm range. 
Positive values means error.

**Promise\<String\> \*Sim800::getNetworkName()**:  
Sends `AT+COPS?` command to the modem and extracts current network operator name.

**Promise\<std::vector\<SMS \*\>\> \*Sim800::getSMSList(SMSListFilter filter = INBOX_UNREAD, bool changeStatus = true)**:  
Sends `AT+CMGL` command to the modem and parse the response.
Possible filters:
  - SMSListFilter::INBOX_UNREAD
  - SMSListFilter::INBOX_READ
  - SMSListFilter::STORED_UNSENT
  - SMSListFilter::STORED_SENT
  - SMSListFilter::SMS_ALL (default)

if you pass `true` to `changeStatus` all of your unread messages marked as read.

**Promise\<uint32_t\> \*Sim800::sendAsciiSMS(const char \*phone, const char \*text)**:  
Sends `AT+CMGS` command to the modem and sends your plain ascii `text` to to `phone`.   
Its safer to format your `phone` into international format (+YYXXXXXXXX where YY is country code).   
If your message sent successfully, it's index in modems memory will accessible as promise's result.

# TODO:
* Add `SMS` & `TextSMS` classes to documention.

# License
The library is licensed under [MIT](LICENSE)