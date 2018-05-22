##1. prerequisites
yum -y install epel-release && yum -y update
yum -y groupinstall "Development Tools"
yum -y install cairo-devel ffmpeg-devel freerdp-devel git java-1.8.0-openjdk libguac libguac-client-rdp libguac-client-ssh libguac-client-vnc libjpeg-turbo-devel libpng-devel libssh2-devel libtelnet-devel libvncserver-devel libwebp-devel libvorbis-devel mariadb-server maven openssl-devel pango-devel pulseaudio-libs-devel terminus-fonts tomcat tomcat-admin-webapps tomcat-webapps uuid-devel wget

systemctl disable firewalld
vi /etc/selinux/config
                SELINUX=disabled
reboot

##2. gucd install
git clone git@github.com:stuntbadger/GuacamoleServer.git
autoreconf -fi
./configure --with-init-dir=/etc/init.d

------------------------------------------------
guacamole-server version 0.9.14
------------------------------------------------

   Library status:
     freerdp ............. yes
     pango ............... yes
     libavcodec .......... no
     libavutil ........... no
     libssh2 ............. yes
     libssl .............. yes
     libswscale .......... no
     libtelnet ........... no
     libVNCServer ........ yes
     libvorbis ........... yes
     libpulse ............ yes
     libwebp ............. yes
     wsock32 ............. no

   Protocol support:

      RDP ....... yes
      SSH ....... yes
      Telnet .... no
      VNC ....... yes

   Services / tools:

      guacd ...... yes
      guacenc .... no
      guaclog .... yes
 
   Init scripts: /etc/init.d
   Systemd units: no

Type "make" to compile guacamole-server.

make && make install && ldconfig

##3.) guacamole client
git clone git@github.com:stuntbadger/GuacamoleClient.git
cd guacamole-client
mvn package
cp guacamole/target/guacamole-0.9.14.war /var/lib/tomcat/webapps/guacamole.war
 
##4.) mysql authentication
mkdir -p ~/guacamole/sqlauth && cd ~/guacamole/sqlauth
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.46.tar.gz

tar -xvf mysql-connector-java-5.1.*.tar.gz
mkdir -p /usr/share/tomcat/.guacamole/{extensions,lib}
mv mysql-connector-java-5.1.*/mysql-connector-java-5.1.*-bin.jar /usr/share/tomcat/.guacamole/lib/
mv ~/guacamole/guacamole-client-*/extensions/guacamole-auth-jdbc/modules/guacamole-auth-jdbc-mysql/target/guacamole-auth-jdbc-mysql-*.jar /usr/share/tomcat/.guacamole/extensions/

##5.) configure database
systemctl restart mariadb.service
mysqladmin -u root password password
mysql -u root -p   # Enter above password
CREATE DATABASE IF NOT EXISTS guacdb DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
GRANT SELECT,INSERT,UPDATE,DELETE ON guacdb.* TO 'guacuser'@'localhost' IDENTIFIED BY 'guacpass' WITH GRANT OPTION;
flush privileges;
quit

##6.) extend database schema
cd ~/guacamole/guacamole-client-*/extensions/guacamole-auth-jdbc/modules/guacamole-auth-jdbc-mysql/schema/
cat ./*.sql | mysql -u root -p guacdb   # Enter SQL root password set above

##7.) configure guacamole
mkdir -p /etc/guacamole/ && vi /etc/guacamole/guacamole.properties

># MySQL properties
>mysql-hostname: localhost
>mysql-port: 3306
>mysql-database: guacdb
>mysql-username: guacuser
>mysql-password: guacpass

# Additional settings
mysql-default-max-connections-per-user: 0
mysql-default-max-group-connections-per-user: 0

ln -s /etc/guacamole/guacamole.properties /usr/share/tomcat/.guacamole/
/root/guacamole-client/extensions/guacamole-auth-totp/target

8.) Google 2fa 
cd /root/guacamole-client/extensions/guacamole-auth-totp/target

cp /root/guacamole-client/extensions/guacamole-auth-totp/target/guacamole-auth-totp-0.9.14.jar /usr/share/tomcat/.guacamole/extensions/
