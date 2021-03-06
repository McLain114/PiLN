# PiLN
Web-based Raspberry Pi Kiln Control Application

This is my first foray into Python, Git, RPi, etc - Please be kind ;-)

I have a good size 220v 45amp electric kiln that has the old "kiln-sitter" style temperature control with bi-metal on/off cycle adjustment knobs. I wanted something with much more precise temperature control for more consistent results and in order to do more work with glass. I started this project with an Arduino board but found the wifi connectivity and coding much too complicated. Being rather proficient in Perl, it seemed the RPi was a better solution. I wrote the original PID control in Perl, but then ported to Python since there seemed to be much more support for it on the RPi and since it was something I wanted to learn anyway.

WARNING! Electricity and heat are dangerous! Please be careful and seek professional help if you are not experienced dealing with high voltage and heat. Use this code/information at your own risk.

This is what I hoped to accomplish:

- Create a PID controller in Python to control an electric kiln. The program takes target temperature, rate, hold time and interval seconds as inputs.
- Daemonize it
- Use MySQL to track temperature change, firing profiles, etc
- Chart the results of firing in real time with google charts
- Wrap in all in a web front end

Many many many thanks to all those who post helpful tidbits out on the web - those on stackoverflow.com in particular. Way to many bits and pieces to give any particular credit. However, I did pull PID calculation code from the following to replace my own code that was iffy (what's math?). It was in C, but was an easy port to Python:

http://brettbeauregard.com/blog/2011/04/improving-the-beginner%E2%80%99s-pid-reset-windup/

All comments, questions, contributions and suggestions welcome!

Future improvements:
- Better calculation of remaining segment time. Currently, remaining time is reset to hold time once temperature is reached.
- Crash/loss of power recovery
- Overheat shutdown

Install:
- Hardware: Raspberry Pi 3, MAX31855 thermocouple interface from Adafruit (https://www.adafruit.com/product/269), High temperature (2372 F) type K thermocouple (http://r.ebay.com/a4cHY1 - search for "kiln thermocouple"), 6 x 40amp Solid State Relays - 2 for each heating element (http://a.co/8PtFgIr), 4 x 20 LCD Display (https://www.adafruit.com/product/198), 5v reed or solid state relay (low voltage load) used to switch 5v supply to high load SSRs (note that the 5v reed relay I used is switched successfully using the 3.3v supply of the GPIO pins), LED for relay on indicator, resistor for LED, variable resistor for LCD contrast adjustment.

- Pin-Out:

		MAX31855+:		3.3v, Pin 1
		MAX31855-:		GROUND, Pin 20
		MAX31855 CLK:		GPIO 25, Pin 22
		MAX31855 CS:		GPIO 24, Pin 18
		MAX31855 DO:		GPIO 18, Pin 12
		REED RELAY+: 		GPIO 4, Pin 7
		REED RELAY-:		GROUND, Pin 20
		REED RELAY INPUT: 	5v, Pin 2
		REED RELAY OUTPUT:	To LED, and high power SSRs (see Fritzing diagram)
		LCD Vss: 		GROUND, Pin 20
		LCD Vcc: 		5v, Pin 2
		LCD Vo: 		Variable Resistor
		LCD RS: 		GPIO 17, Pin 11
		LCD R/W: 		GROUND, Pin 20
		LCD E:			GPIO 27, Pin 13
		LCD DB4:		GPIO 12, Pin 32
		LCD DB5:		GPIO 16, Pin 36
		LCD DB6:		GPIO 20, Pin 38
		LCD DB7:		GPIO 21, Pin 40
		LCD LED+:		5v, Pin 2
		LCD LED-:		GROUND, Pin 20
		

- Install PiLN files in /home and create log directory:

		cd /home
		sudo git clone https://github.com/pvarney/PiLN
		sudo mkdir /home/PiLN/log
		
- Install MySQL/PHPMyAdmin (PHPMyAdmin not required):

		sudo apt-get install mysql-server
		sudo apt-get install mysql-client php5-mysql
		sudo apt-get install phpmyadmin
		
- Set up Apache for PHPMyAdmin if required. Edit /etc/apache2/apache2.conf and add the follow at the bottom of the file:
	
		Include /etc/phpmyadmin/apache.conf
		
- Set up directories/link for web page:

		sudo mkdir /var/www/html/images	
		sudo mkdir /var/www/html/style	
		sudo ln -s /home/PiLN/images/hdrback.png /var/www/html/images/hdrback.png	
		sudo ln -s /home/PiLN/images/piln.png /var/www/html/images/piln.png	
		sudo ln -s /home/PiLN/style/style.css /var/www/html/style/style.css
	
- Add the following ScriptAlias and Directory parameters under "IfDefine ENABLE_USR_LIB_CGI_BIN" in /etc/apache2/conf-available/serve-cgi-bin.conf:
	
		ScriptAlias /pilnapp/ /home/PiLN/app/
		  <Directory "/home/PiLN/app/">
		    AllowOverride None
		    Options +ExecCGI -MultiViews +SymLinksIfOwnerMatch
		    Require all granted
		  </Directory>

- Create links to enable cgi modules:
	
		cd /etc/apache2/mods-enabled
		sudo ln -s ../mods-available/cgid.conf cgid.conf
		sudo ln -s ../mods-available/cgid.load cgid.load
		sudo ln -s ../mods-available/cgi.load cgi.load

- Restart Apache:
	
		sudo systemctl daemon-reload
		sudo systemctl restart apache2
		
- Install required Python packages:
		
		sudo apt-get install python-mysqldb
		sudo apt-get install python-dev
		sudo pip install PyMySQL
		
- Install Adafruit MAX31855 Module:

		cd ~
		git clone https://github.com/adafruit/Adafruit_Python_MAX31855.git
		cd Adafruit_Python_MAX31855
		sudo python setup.py install		
		
- Required Python modules (separate installs were not required for these using the latest Raspian build as of July 2017): cgi, jinja2, sys, re, datetime, pymysql, json, time, logging, RPi.GPIO.

- Log into the MySQL command line, create the database, create the user and give permissions, then load PiLN.sql to build table structures:

		mysql -uroot -p
		mysql> create database PiLN;
		mysql> CREATE USER 'piln'@'localhost' IDENTIFIED BY 'p!lnp@ss';
		mysql> GRANT ALL PRIVILEGES ON PiLN.* TO 'piln'@'localhost';
		mysql> use PiLN;
		mysql> source /home/PiLN/PiLN.sql;

- To enable automatic startup of the daemon (Had to do the copy/enable/delete/link in order to get systemctl enable to work):

		cp /home/PiLN/daemon/pilnfired.service /etc/systemd/system/
		sudo systemctl daemon-reload
		sudo systemctl enable pilnfired
		sudo rm /etc/systemd/system/pilnfired.service
		sudo ln -s /home/PiLN/daemon/pilnfired.service /etc/systemd/system/pilnfired.service
		sudo systemctl daemon-reload
		sudo systemctl start pilnfired
		sudo systemctl status pilnfired
	I also had to convert mysqld to start up from systemd so that I could set a "want" for pilnfired (mysqld.service file from https://gist.github.com/thomasfr/e4e4bb64352ee574334a):
	
		cp /home/PiLN/daemon/mysqld.service /etc/systemd/system/
		sudo systemctl daemon-reload
		sudo systemctl enable mysqld
		
- Tuning: I spent a while adjusting the PID parameters to get the best results and am still tuning. Your tuning parameters will depend on your specific application, but I used the following which might be a good starting point:

		Proportional:	6.00
		Integral:	0.04
		Derivative:	0.00
	I also had good results setting the interval seconds to 10

Using the Web App:
- On the same network that the RPi is connected to, go to http://<RPi_IPAddress>/pilnapp/home.cgi
From there, I hope it's all fairly self-explanatory. Feel free to contact me with questions!

UPDATE 4/8/2018:
I had a problem with one of the heating elements not working and found that the SSRs couldn't hold up to the heat - even with the heat sinks. I have since replaced them with mechanical relays (same type that were in it before I ripped the guts out). I am now using a 4 relay module to switch on/off the 220v coil voltage on the relays. I also put the MAX31855 module in its own metal box to see if it would reduce the amount of temperature fluctuation. 
