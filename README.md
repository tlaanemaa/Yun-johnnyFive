<h1>How to set up NodeJS and johnny-five on Arduino Yun</h1>

<p>It took me about 2-3 days to get johnny five properly working on my Arduino Yun. so I thought Id write down the steps I took so others (and myself in future) can use it when facing similar task.</p>


<h3>Preparing Yun</h3>
First you need to make sure your Yun's firmware is up to date. This should be done first because updating firmware deletes all files on your Yun. There is a good instruction on how to do this at: <a>http://arduino.cc/en/Tutorial/YunSysupgrade</a>

Next you'll want to increase your Yun's storage space because the flash memory that comes with Yun is not very large and it also wears out overtime. So using an SD card both gives you more space to work with and also somewhat extends the life of your Yun. Good instructions for that can be found at: <a>http://arduino.cc/en/Tutorial/ExpandingYunDiskSpace</a>.
There is also a newer version of the disk space expander sketch that they mentioned in the instruction, the new one should laos create a swap partition. I havent tried it myself but there's a link to where to get it at the links section if you want to try.

<h3>Installing NodeJS and node-serialport</h3>
Now its time to SSH into your Yun and install NodeJS. If you are on a windows machine (like I was) then <a href="http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html">Putty</a> is a good SSH client to use. Installing NodeJS is easy, just type the following commands:
<pre>opkg update</pre>
<pre>opkg install node</pre>
<i>Installing NodeJS (the second command) took a looooong time (~30-45 min maybe) for me so be patient.</i><br>

Next up, you'll need to install node-serialport. This is needed to facilitate the connection between the OpenWRT linux side of Yun and the Arduino ATmega 32U4 side. To do that, type in the following commands:
<pre>opkg update</pre>
<pre>opkg install node-serialport</pre>
<i>Same here, installing node-serialport took a long time</i><br>

Next you'll need to disable the bridge script so NodeJS can use that channel instead. To do that edit the <code>/etc/inittab</code> file and comment out the <code>ttyATH0</code> line. You can edit the file by typing
<pre>nano /etc/inittab</pre>
The file should look something like this after editing:
<pre>
::sysinit:/etc/init.d/rcS S boot
::shutdown:/etc/init.d/rcS K shutdown
#ttyATH0::askfirst:/bin/ash --login
</pre>

<h3>Uploading Firmata sketch</h3>
Next upload the StandardFirmataYun sketch to your Arduino, you can do this over wifi or with an USB cable, doesnt matter. The sketch can be found here: <a>https://github.com/firmata/arduino/tree/master/examples/StandardFirmataYun</a> and I also have it available in this repo <a href="https://github.com/tlaanemaa/Yun-johnnyFive/blob/master/StandardFirmataYun.ino">here</a>. Arduino IDE comes with a version of StandardFirmata but it has problems that make it not suitable for Yun (namely not using ATH0 and not handling the u-boot problem, see the links section). 

<h3>Preparing your project</h3>
Now the last thing left to do is to install the <a href="https://github.com/rwaldron/johnny-five">johnny-five</a> module and all other modules you need for your project (I also used socket.io and express). You'll probablly need to do this on your computer since Yun doesnt have enough RAM to install npm modules (you get an <i>out of memory</i> error).
To install those modules simply navigate to your projects's folder in your PC and type in:
<pre>npm install module-name</pre>
So to install johnny-five type 
<pre>npm install johnny-five</pre>
Before copying the modules over to your Yun make sure to delete the <code>serialport</code> module from <code>johnny-five/node-modules</code> folder. This is needed because serialport on your PC is compiled for your PC and may not work on Yun. If we delete it then johnny-five will use the node-serialport we installed on Yun earlier.
I used <a href="http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html">PSCP</a> to upload my project to my Yun.

When defining the board in johnny-five make sure to pass the port argument. It didnt work otherwise for me.
<pre>var board = new five.Board({port: "/dev/ttyATH0"});</pre>

<h3>Done!</h3>
Thats it. Everything should work fine now.
Below is the server script I used on Yun to turn an LED on and off from a website. It uses express to host the webserver, socker.io to send information from the website to NodeJS and johnny-five to control the led. Same setup should work with any other johnny-five capabilites (motors, servos, sensors, etc.)

<pre>
// Basic web server setup
var express = require('express');
var srv = express();
var http = require('http').Server(srv);
var io = require('socket.io')(http);
var port = 8080;

srv.use(express.static(__dirname));

http.listen(port, function(){
  console.log('listening on localhost:' + port );
});

// Johnny-five setup
var five = require('johnny-five');
var board = new five.Board({port: "/dev/ttyATH0"});
var boardReady = false
var led

board.on("ready", function() {	
	//Set Hardware variables
	led = new five.Led(13);
	
	// Write down that board is ready
	boardReady = true;
	console.log("Board ready!");
});

// Start listening for socket events
io.on('connection', function(socket){

	// Led state changed
	socket.on('led', function(data){
		if(boardReady){
			if(data == true) led.on();
			else led.off();
		}
	});
	
});
</pre>


<h3>Links</h3>
<ul>
<li>A guide on the same subject that helped me: <a>http://cylonjs.com/documentation/platforms/yun/</a></li>
<li>NodeJS: <a>http://nodejs.org/</a></li>
<li>Johnny-Five: <a>https://github.com/rwaldron/johnny-five</a></li>
<li>Socket.io: <a>http://socket.io/</a></li>
<li>Express: <a>http://expressjs.com/</a></li>
<li>Arduino Yun: <a>http://arduino.cc/en/Main/ArduinoBoardYun?from=Products.ArduinoYUN</a></li>
<li>Another guide on expanding disk space and installing NodeJS: <a>http://blog.arduino.cc/2014/05/06/time-to-expand-your-yun-disk-space-and-install-node-js</a></li>
<li>Another article on this: <a>https://github.com/born2net/mediaArduino</a></li>
<li>A forum post about how firmata blocks yun's boot: <a>http://forum.arduino.cc/index.php?topic=215338.msg1577312#msg1577312</a></li>
<li>New version of disk space expander that also allows creating swap: <a>https://github.com/Fede85/YunSketches/tree/master/YunDiskSpaceExpander</a></li>
<li>Official firmata for Yun: <a>https://github.com/firmata/arduino/tree/master/examples/StandardFirmataYun</a></li>
</ul>
