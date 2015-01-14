### About
This is a guide to install the Open GPS Tracking System on Ubuntu 14.04. I used Kubuntu 14.04 and Ubuntu Server 14.04 in my tests, but the installation is very likely to work fine on newer and (not too) older versions as well. Probably a few changes can make it work on Debian.

Further configuration of Apache, Tomcat and security issues are beoynd the scope of this guide. My aim is to provide a functional OpenGTS system following the [official manual] procedures.

The text is copy-and-paste friendly. Whenever you see a `VER` variable, you are expected to visit the provided link and replace the current number by the most recent version of the package.


##### LAMP, ant and unzip
```bash
sudo apt-get update
sudo apt-get install apache2 php5 mysql-server libmysql-java ant unzip

sudo /etc/init.d/mysql start
```

##### Java
You may want to use the Sun's version, as the [official manual] recommends. As many people, I haven't had any problem using OpenJDK.
```bash
sudo apt-get install openjdk-7-jdk
export JAVA_HOME=/usr/lib/jvm/java-7-openjdk-amd64
echo "export JAVA_HOME=/usr/lib/jvm/java-7-openjdk-amd64" >> ~/.bashrc
```

##### tomcat
Check the current version at http://apache.mirror.uber.com.au/tomcat/tomcat-7/
```bash
VER=7.0.57

cd /tmp
wget -c http://apache.mirror.uber.com.au/tomcat/tomcat-7/v${VER}/bin/apache-tomcat-${VER}.zip
unzip apache-tomcat-${VER}.zip
sudo cp -a apache-tomcat-${VER} /usr/local/

export CATALINA_HOME=/usr/local/apache-tomcat-${VER}
cd /usr/local
sudo ln -s $CATALINA_HOME tomcat
cd $CATALINA_HOME/bin
chmod a+x *.sh
$CATALINA_HOME/bin/startup.sh
echo "export CATALINA_HOME=/usr/local/apache-tomcat-${VER}" >> ~/.bashrc
```


##### Java Connector
Check the latest version at http://dev.mysql.com/downloads/connector/j/#downloads
```bash
VER=5.1.34

cd /tmp
wget -c http://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-${VER}.zip
unzip mysql-connector-java-${VER}.zip
cd mysql-connector-java-${VER}
sudo cp mysql-connector-java-${VER}-bin.jar $JAVA_HOME/jre/lib/ext
```

##### Java mail
Check the latest version at https://maven.java.net/content/repositories/releases/com/sun/mail/javax.mail/
```bash
VER=1.5.2

cd /tmp
wget -c https://maven.java.net/content/repositories/releases/com/sun/mail/javax.mail/${VER}/javax.mail-${VER}.jar
sudo cp javax.mail-${VER}.jar $JAVA_HOME/jre/lib/ext/.
sudo mv $JAVA_HOME/jre/lib/ext/javax.mail-${VER}.jar $JAVA_HOME/jre/lib/ext/javax.mail.jar
```


##### OpenGTS
curl can select a mirror and download OpenGTS. You can use your browser and download on [SourceForge](http://sourceforge.net/projects/opengts/files/latest/download) as well.
Choose the appropriate user and group.
```bash
sudo apt-get install curl

cd /tmp
VER=2.5.7
curl -L http://downloads.sourceforge.net/project/opengts/server-base/${VER}/OpenGTS_${VER}.zip > OpenGTS_${VER}.zip
sudo unzip /tmp/OpenGTS_${VER}.zip -d /usr/local

GROUP=users
sudo chown -R ${USER}:${GROUP} /usr/local/OpenGTS_${VER}
export GTS_HOME=/usr/local/OpenGTS_${VER}
echo "export GTS_HOME=/usr/local/OpenGTS_${VER}" >> ~/.bashrc
```


##### env variables & symlinks

```bash
echo "export ANT_HOME=/usr/share/ant" >> ~/.bashrc
source ~/.bashrc

sudo ln -s $JAVA_HOME /usr/local/java
sudo ln -s $CATALINA_HOME /usr/local/tomcat
sudo ln -s $GTS_HOME /usr/local/gts
```

##### Basic configuration
We will uncomment the lines related to the database's user and password in `config.conf`.

```bash
sed -i "s/#db.sql.user=gts/db.sql.user=gts/" $GTS_HOME/config.conf
sed -i "s/#db.sql.password=opengts/db.sql.password=opengts/" $GTS_HOME/config.conf
```

If the following folder points to itself, unlink it.
Verify whether $CATALINA_HOME has a folder apache-tomcat-${VER} that points to itself.
```bash
ls -l $CATALINA_HOME
```
If it the recursive link exists, unlink to avoid problems on the OpenGTS compilation.
```bash
unlink /usr/local/apache-tomcat-${VER}/apache-tomcat-${VER}
```
##### Compilation & initialisation
Finally, let's compile OpenGTS.
```bash
cd $GTS_HOME
ant all

# If your password has special characters, enclose it with quotes
bin/initdb.sh -rootUser=<rootUser> -rootPass=<rootPass>
```

Check whether everthing's correct.

```bash
cd $GTS_HOME && bin/checkInstall.sh
```

Add account, install Track Java Servlet
```bash
bin/admin.sh Account -account=sysadmin -pass=password -create

cd $GTS_HOME && ant track
cp build/track.war $CATALINA_HOME/webapps/.
```

You can now test the site on http://localhost:8080/track/Track .

If it is not accessible, you may have to start (or restart) Tomcat.
```bash
# Check whether tomcat is running.
ps -ef | grep tomcat

# If it is running, execute the lines below.
$CATALINA_HOME/bin/shutdown.sh
rm -rf $CATALINA_HOME/webapps/track*
cp $GTS_HOME/build/track.war $CATALINA_HOME/webapps/.
$CATALINA_HOME/bin/startup.sh

# Else
$CATALINA_HOME/bin/startup.sh
```

Install Event Java Servlet and gprmc.
```bash
cd $GTS_HOME && ant events
cp -v build/events.war $CATALINA_HOME/webapps

cp build/gprmc.war $CATALINA_HOME/webapps/.
```

Verify installation again:
```bash
bin/checkInstall.sh
```


##### Client
If OpenGPS installation was successfull, you may want to test it quickly by monitoring an Android mobile phone.

1. [Download](https://play.google.com/store/apps/details?id=org.opengts.client.android.cgtsfre&hl=en) CelltracGTS, the [Android official client].
2. Make sure that the server has the gprmc module and it is working:
First, check whether the installation is correct:
```bash
cd $GTS_HOME && bin/checkInstall.sh
```
Then, verify whether the module is working:
```bash
cd $GTS_HOME && cat logs/w-gprmc.log
```
The file above should exist.
3. On CelltracGTS app, go to settings and configure "Server URL" as that, making the values reflect your server configuration.
`http://DOMAIN:PORT/gprmc/Data?`
4. In OpenGTS, go to `Administration > Vehicle Admin` and insert the Unique ID field as
`gprmc_MOBILEID`
where MOBILEID is the "Mobile-id" number in CelltracGTS Settings.

From the official documentation:
>Once the server has been set up, you can send a test message from the phone by selecting the status code "Waymark" and clicking "Send". If everything has been properly configured, the "Event Status" section on the phone should first show "Queue: 1 Sent: 0", followed by "Queue: 0 Sent: 1" as the event is sent to the server."

You should be able to see track the phone using OpenGTS by accessing `Mapping > Vehicle`.

### Troubleshooting
I tested these procedures on two clean Ubuntu 14.04 installations, but if you are installing OpenGTS on servers that were already in use, there might be issues such as firewall configuration and port conflicts.


### To-do
* Restart tomcat automatically on boot
* http://opengts.sourceforge.net/FAQ.html#faq_htmlFrame
* (maybe) edit tomcat-users.xml

### Credits
Most of the instructions are based on the OpenGTS [official manual].

Copy and paste friendly style is inspirated from [opengts-server-install-step-by-step.org](https://github.com/troywill/opengts-android/blob/master/opengts-server-install-step-by-step.org) (for Arch Linux).

[official manual]:http://opengts.sourceforge.net/OpenGTS_Config.pdf
[Android official client]:http://www.geotelematic.com/CelltracGTS/Free.html
