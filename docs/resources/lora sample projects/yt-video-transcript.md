00:00:03
What if I tell you you can send messages to someone kilometers away without having any kind of network or even a SIM card? In this video, I'm going to show you exactly how to do that by building this [Music] This is our very own off-the-g grid communication device built using the ESP32S3 and the SX1262 Laura module powered by Meshtastic which is an open-source mesh networking firmware that allows device to talk to each other. In this video, I'm going to show you how these devices work and

00:00:50
explain the circuit diagram and show you how to upload the firmware so that you can build it on your own. Before we start building, let me quickly introduce the sponsors for this video. Digi. Digi is a global leader in cuttingedge commerce distribution of electronic components and automation products worldwide. They provide more than 16.5 million components from over 3,000 manufacturers with products in stock available for immediate shipment. Also, with their fast shipping and excellent customer support, you can always trust

00:01:20
that your products will arrive on time and in top condition. So do remember to check out Digi Key for your next project. Okay. So for those who are new to mestastic, it is a LoRa based communication platform which is completely open-source and this is how it works. This is a typical LoRa mesttastic network. The most important thing you can see here are these nodes which is connected together like a mesh network. So these nodes are what we are building in this video. So these nodes can do two things. one, it can talk to

00:01:51
other nodes using LoRa and it can also talk to a mobile phone using Bluetooth. The mobile phone will be running the mesttastic application or it can also talk to a desktop using Bluetooth, Wi-Fi or serial communication. Now, in our case, we are building this node using the ESP32 for its Wi-Fi and Bluetooth capabilities and we are using the SX1262 LORA module from seed studio for LoRa communication. Now what you can see here is since all these nodes are connected in a mesh network if this node wants to

00:02:21
send a message to this node and it is really outside the range the message can get hopped between other nodes which are also in the meshtastic node. That way you can form an ecosystem and send messages for really long distances. Now these nodes can also be purchased off the shelf. There are a lot of vendors who sell it as a node but in this project we are going to show you how to build it on your own so that it's more cost effective and much more fun to use. So now let's take these two nodes

00:02:47
outside so that I can show you how it works. Okay, here one node is placed on the rooftop of our office building connected to this phone. My other node is placed in a card connected to a different phone from which I will be sending messages as I travel around. As you can see here, it works really well up to a range of 1 and 1/2 kilometers. But after that it is not very reliable and I had to roll down my windows to make sure the messages get delivered. So yeah based on what we tested it is proven that these nodes can

00:03:21
communicate to a distance of 1 and 1/2 kilometers without any problem. Now do keep in mind that we tested node to node communication but in an actual meshtastic network there will be a cluster of devices connected together and the messages would hop around from one device to another extending your total range. That being said, now let's start building our very own meshtastic node. As you can see here, the circuit diagram is very simple. Even our hardware is very simple. It just have a PCB board with four vital components, a

00:03:51
battery to keep the whole thing powered and an antenna for Laura communication. The whole thing is put up inside this enclosure to keep everything compact. Now coming back to the circuit diagram, let's start from the top left. The first thing over here is a type-C USB port. So this USB port over here has multiple functionalities. The first thing is it is used to program our ESP32 module on board and then it is also used for serial communication with our ESP32 and then it is also used for charging our

00:04:24
lipo battery on board. So what we have done here is that we have connected the data pins of the USB to the data pins of ESP32 and then we have connected the Vbus to our power supply section here which we will get back to later and then we have few TVS diodes for search protection. Moving on we have the ESP32S3 which is the main brain behind our LoRa node over here. So this IC here is called the ESP32S3W room. It is from Expressive Systems and this thing has both Wi-Fi and Bluetooth capabilities. Like I told you earlier,

00:05:02
we're going to use this Wi-Fi and Bluetooth functionality to be able to communicate with the Mesttastic mobile application or the Mesttastic web application. Moving on, we have the user button section where you can see two buttons over here. So one button is the boot button for the ESP32S3 microcontroller and the other is a reset button. Apart from that you can also notice that we have a voltage divider over here which can be used to measure the voltage of this battery using the ADC pin of ESP32 microcontroller. So

00:05:34
this module is actually called the VOX1262 wireless module from seed technology. The best thing about using a module is that seed has done a very good job with packaging this LoRa IC and having all the certifications. So it's very easy to use this module for LoRa communication. You cannot just use this for mestastic but for any LoRa communication this would be a perfect choice. Moving on, we have the power supply section over here which has two important IC's. The first IC is the LTC 4054

00:06:08
which is from analog devices. So what it does is it takes the voltage from the USBC and it makes sure that the voltage is sent appropriately for our battery to charge and once the battery is fully charged it automatically stops the charging process. So over here we have a blue LED connected to the charge pin to indicate if the battery is being charged and then we have also set the charging current using the 2 kiloohms resistor over here. So this particular IC can do a maximum of 800 milliamps charging

00:06:41
current and it is perfect for single cell lithium ion or lithium polymer batteries. The next IC over here is the ADP124 which is again from analog devices. So this is a 3.3 volt linear voltage regulator IC in a very small form factor perfect for portable devices like this. So the maximum input wtage for this IC is 5.5 volt and the regulated output is 3.3 volt which is needed for our ESP32 and our SX1262 LOA module. So this IC has a low dropout voltage as well. It's just 0.23 volts at 500 milliamps and the

00:07:19
maximum output current for this IC is 500 milliamps. Now over here we have few other ports which is not used in this project but it is kept like an expansion port. So if you want to add an OLED display to your node or add more inference, you can use this thing. The complete circuit diagram for this project along with the PCB design files and the firmware can be found on our website. The link for the same is given in the description. You can just visit the link, download all the required files, fabricate your own PCBs, buy your

00:07:50
components and just solder them together to build your own meshtastic note. As a bonus, we have also provided a 3D design so that you can print a neat enclosure for your board. Just like this. Now that you have your boards ready, let's get back to the computer so that I can show you how you can upload the firmware. We have made a PCB board for the same circuit diagram. And this is how our PCB looks like. We have also marked the different parts on our board for you to understand it easily. And the complete

00:08:18
explanation along with how the PCB is designed is also shown over here. Now once your hardware is ready, you can proceed with uploading the code. To do that, head over to the link in the description and click on this button which says code and schematics. Inside here, you will find a lot of files. This is a custom firmware that we have modified for this particular hardware. If you're interested, you can go through the code files and see how everything is done. But if you just want to upload the

00:08:50
code, the easiest way to do that is open this GitHub link and click on the binary files. And as you can see we have three binary files over here. Just download all these three file. Connect your board to your computer using the ESP type C connector available. You can also see the blue LED glowing to indicate that your battery is charging. So once you're connected you can go ahead and search for ESP flasher tool and you should see something like this. And then uh before you can start uploading the code, make

00:09:24
sure you reset this board so that the port is available for programming. Just hold on the reset button and then press the boot button. You should be able to see your device on your computer when you click connect. So mine is called the USB J tag. You should also be finding some name similar. Click on connect and then you'll get an option to upload the bin files for you to program your ESP32. So this is that directory. Over here you have to get into the binary files folder and then check out this readme where we

00:09:59
have mentioned the address and the respective uh file name which you should add. So there are three addresses here. Let's do all that over here. So yeah, make sure the address mentioned in your readme and the respective file name is matched like it is shown over here and then click on program. So now my code is completely uploaded and once it is done we can just remove the power and connected again to just restart our board. So yes now the green LED has started to blink. This means we are ready to use the meshtastic firmware

00:10:32
which we just uploaded. So to do that just get into meshtastic.org and then over here you will find an option to connect your device. At step three just click connect your device and then try out the web client. So then click on new connection. We have already connected it to our computer using serial. So we'll be doing the serial. Click on this thing and then you will see the ESP32 name over here. If this is the first time that you're doing this, make sure to get into configuration and

00:11:01
over here under LORA make sure to set the region to your country. Once that is done, if there are any other LoRa modules near you, in my case I have another Lora mestastic node which is the same thing. So we should be able to send messages to and fro. Let me quickly show you how to do that. Okay. So what I have done here is it's the same setup. We have two LoRa modules over here. This thing in my hand is called CD2. As you can see on the desktop web app, it says CD2. So this device called CD2 is

00:11:34
connected to my computer using a type-C cable and it is connected on this web application as you can see here. Now if I get into messages on the right side I can see this device is finding one more device called CD1 which is the another node right next to me. Now in order to communicate between these two nodes I'm going to connect this to my mobile phone to show you how the data sent from this node is reaching the other node. So to do that I have my mobile phone screen mirrored over here. I'm just going to

00:12:04
open the meshtastic app. So as you can see it's already connected and subscribed. the Bluetooth signal everything is shown over here. If we go into the node section, we can see all the LoRa nodes that's available nearby. So this device which is connected to phone when Bluetooth is called CD1. The only other device near me is CD2 which is connected to my desktop. And as you can see it says signal strength is like really good because it's literally right next to. So now to send messages let's

00:12:33
click on this thing. You can see that it is a private hardware because we have built it on our own. So there's a lot of other options as well. You can explore it if you're interested. You can even send GPS coordinates. You can use the GPS coordinates of your mobile to know where is your module and where the other nodes in your network are. So then it has a lot of uh other useful features. So test it out if you're interested. I'm just going to click on messages here. As you can see uh these are some tests

00:13:00
which we have done earlier. Now let me quickly send a hi from here. And once I do that you can see on my desktop this messages has become one. I'm just going to go here and click on the cd mesh one. So you can see it has sent hi. This message has also been acknowledged and we have received it here. So let's do another test from here and as soon as I send a message from here it is received on my phone as well. So yeah, this is how uh the messages can be sent between two LoRa nodes. You can either connect

00:13:33
it to a computer or connect it to a mobile phone. In this test demo, one is connected to a computer using USB serial, but you can connect both of these to a mobile phone as well with blue. So yeah, this is how you can build a Nora node using mtstic and this is how you can communicate. With that, we have come to the conclusion of this video. Hope you learned something useful and enjoyed watching it. That being said, this is Ashwin from Circuit Digest signing off. Tata. Bye-bye.

