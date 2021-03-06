# Sources of Hadoop Summit Cool-Tech Keynote
These sources were used at Hadoop Summit Cool-Tech Keynote.

The demo interacted with the audience by letting them vote on a question with
two possible options (A or B). Each attendee was able to vote by sending out a
text over SMS or Twitter which contained a hashtag according to the vote
(#A or #B). Lightboxes on stage indicated each incoming vote with a flashlight
and returned environmental sensor-data back to the phone of a participant.

The overall demo consists of several technical components:

1. [Twilio](https://www.twilio.com/) as the SMS provider for sending and receiving SMS to/from the audience.
2. [Twitter](https://www.twitter.com/) as another inbound channel for receiving votes from the audience. GetTwitter processor of NiFi leverages the Twitter Streaming API to listen on Tweets tagged with *#HadoopSummitDemo*
3. [Apache NiFi](https://nifi.apache.org/) as the central orchestration tool. The dataflow used for the demo is
introduced below and is available in the Repo as a [template](./nifi/voting-dataflow-template.xml)
4. A MySql database to store the votes of all participants
5. Two Lightboxes as IoT edge devices on stage. These boxes are selfmade.
Below you can find the detailed setup of these boxes. The entire software solution on the Lightboxes is provided
in this Git-Repo (see __./rpi__ folder) along with instructions on how to run it on your own box.
6. [AWS IoT](https://aws.amazon.com/iot/) used as the MQTT broker for communicating with the Lightboxes

# Voting dataflow powered by Apache NiFi

1. Install [Apache NiFi](https://nifi.apache.org/) (e.g. on an EC2 instance of AWS)
It's straightforward.
2. The demos use custom processors for AWS IoT which are currently not part of the
base package. You have to pull my [fork](https://github.com/KayLerch/nifi/tree/nifi-aws-iot-processor)
of NiFi and build and deploy the AWS processor bundle to your NiFi instance.

        cd {nifi-fork-home}/nifi-nar-bundles/nifi-aws-bundle
        mvn -T 2.0C clean install
        cp ./nifi-aws-nar/target/nifi-aws-nar-x.y.z-SNAPSHOT.nar ~/{nifi-home}/lib

3. Go to NiFi's admin page and import the [template](./nifi/voting-dataflow-template.xml) you can find in __./nifi/__ folder
4. Adjust configuration in the processors as sensitive information does not ship
with the template. Moreover, the linked controller services need to be created or
updated with credentials to e.g. AWS.

![NiFi Dataflow](./nifi/voting-dataflow.png)

# Voting database
Set up a MySql instance e.g. on AWS RDS and create this table:

    CREATE TABLE `voting` (
      `participantId` int(11) NOT NULL,
      `senderAlias` text,
      `text` text,
      `votedOnA` bit,
      `votedOnB` bit,
      `votedAt` TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    ) ENGINE=InnoDB DEFAULT CHARSET= latin1;

# Twilio SMS
Once you registered to [Twilio](https://www.twilio.com/) and set up a messaging service in "Programmable SMS". Add
an HTTP endpoint which points to your NiFi instance. The HTTP processor of this demo's
dataflow listens on port 8081.

# Lightboxes
The edge devices in the voting demo were used to indicate votes of the
audience with flashlights on 32x32 RGB LED Matrix. Moreover, they measure
temperature and humidity of the nearby environment to send it back to
the attendees. The overall solution consists of two Lightboxes where
each of them is dedicated to one of the two options of the voting.

![Lightbox](./rpi/img/lightbox.png)

## Physical set-up
If you want to rebuild the demo device you can start with following equipment:

![Setup](./rpi/img/lightbox-wiring.png)

* (A) [32x32 RGB LED Matrix Panel - 6mm pitch](https://www.adafruit.com/products/1484)
* (B) [Female DC Power adapter - 2.1mm jack to screw terminal block](https://www.adafruit.com/products/368)
* (C) [USB to 2.1mm Male Barrel Jack Cable - 22AWG & 1 meter](https://www.adafruit.com/products/2697)
* (D) [Raspberry Pi 3 - Model B - ARMv8 with 1G RAM](https://www.adafruit.com/products/3055)
* (E) [Half-size breadboard](https://www.adafruit.com/products/64)
* (F) [USB cable - 6" A/MicroB](https://www.adafruit.com/products/898)
* (G) [Assembled Pi T-Cobbler Plus - GPIO Breakout - for RasPi A+/B+/Pi 2/Pi 3](https://www.adafruit.com/products/2028)
* (H) [Pi Model B+ / Pi 2 / Pi 3 Case Base - Clear](https://www.adafruit.com/products/2253)
* (I) [DHT22 temperature-humidity sensor](https://www.adafruit.com/product/385)
* (J) [Premium Male/Male Jumper Wires - 20 x 6" (150mm)](https://www.adafruit.com/product/1957)
* (K) [Premium Female/Male Jumper Wires - 40 x 6" (150mm)](https://www.adafruit.com/product/826)
* (L) [Dual USB External Battery, Premium Power Bank High Capacity Most Compact Portable Charger](http://www.amazon.com/HyperGear-16000mAh-External-Capacity-Portable/dp/B0188WLTI8/ref=sr_1_8?ie=UTF8&qid=1464979881&sr=8-8&keywords=powerbank+4.6A) (or similiar)

![Setup](./rpi/img/breadboard.png)

How to wire the LED matrix to the J8 header of a Raspberry Pi is explained in this [wiring diagram](https://github.com/hzeller/rpi-rgb-led-matrix#wiring-diagram)
If you're not familiar with the J8 pins [this](http://raspi.tv/wp-content/uploads/2014/07/Raspberry-Pi-GPIO-pinouts.png) might help you.

## Software set-up

Solution is implemented and runs with Python. Sources can be found in __./rpi__ folder.

The Python scripts rely on __Python 3.4+__ and meant to run as
daemons on a Pi. Once started they turn your Pi into an IoT
edge computing device.

1. Be sure you have the latest __[Raspbian image](https://www.raspbian.org/RaspbianImages)__ on your Pi

2. Install __Paho MQTT__ for communitcating with AWS IoT over MQTT

        sudo pip3 install paho-mqtt

3. Install __Adafruit DHT__ libraries to read out temperature and humidity from a DHT22 sensor connected to a GPIO pin of your Pi

        git clone https://github.com/adafruit/Adafruit_Python_DHT.git
        cd Adafruit_Python_DHT
        sudo python3 setup.py install

4. Install Python libraries to work with __Adafruit's RGB Matrix__ which is also wired to several pins of our Pi.

        git clone https://github.com/hzeller/rpi-rgb-led-matrix.git
        cd rpi-rgb-led-matrix/python
        sudo python3 setup.py build
        sudo python3 setup.py install

    You may be faced with an error while building these libraries. See this [Bugfix](https://github.com/Saij/rpi-rgb-led-matrix/commit/d61b537478f418038e9bfdf67ca6932fb0f39fa9) on Github

5. Now bring the demo code to your Pi

        git clone https://github.com/KayLerch/hadoop-summit-cooltech-voting.git

6. Go to management console of __AWS IoT__, register a thing and generate __TLS certificates__.
The certificates you are provided with need to be stored in __./cert__ like this:

        ./cert
            ./{thing_name}
                ./certificate.pem.crt
                ./private.pem.key

    where *{thing_name}* is the name of your thing as registered in AWS IoT.

## Run
### Temperature
This script acts as an MQTT client. It periodically reads out Adafruit DHT22
temperature sensor for current temperature and humidity and publishes data to
an MQTT topic of your thing in AWS IoT. The dataflow of Apache NiFi will read
from this thing's shadow device and gets the last reported temperature and
humidity without being connected or dependant on the edge devices.

You have to provide at least three parameters

    -t | --thing {thing_name}

where *{thing_name}* is the name of your thing as registered in AWS IoT.

    -p | --pin {pin_no}

where *{pin_no}* is the GPIO pin number the DHT22 sensor is connected to. Default is 21.

    -e | --endpoint {awsiot_endpoint}

where *{awsiot_endpoint}* is the account-specific endpoint to you AWS IoT gateway service.

    -v | --verbose

This will print out runtime information to console. However, you can have a look into __./logs/temperature.log__
at any time without using verbose argument.

This is an example:

    sudo python3 temperature.py -t myRPi -p 6 -e A1B71MLXKNXXXX.iot.us-east-1.amazonaws.com

### Voting
This script acts as another MQTT client. It subscribes to an MQTT topic of a thing
you registered in AWS IoT. The voting dataflow in Apache NiFi will feed this topic
with the votes coming in as SMS or Tweets to NiFi. Once a vote is received this
script let the RGB matrix flash a light and increment the displayed counter.

You're asked to provide at least two parameters

    -t | --thing {thing_name}

where *{thing_name}* is the name of your thing as registered in AWS IoT.

    -e | --endpoint {awsiot_endpoint}

where *{awsiot_endpoint}* is the account-specific endpoint to you AWS IoT gateway service.

    -v | --verbose

This will print out runtime information to console. However, you can have a look into __./logs/voting.log__
at any time without using verbose argument.

This is an example:

    sudo python3 voting.py -t myRPi -e A1B71MLXKNXXXX.iot.us-east-1.amazonaws.com
