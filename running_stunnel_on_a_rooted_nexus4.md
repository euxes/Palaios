Running Stunnel on a Rooted Nexus 4
===================================

This method probably works on other __rooted__ Android devices, I only have Nexus 4 tested.

Prerequisites
-------------

First, you have to get your Nexus 4 rooted. And make sure having __Busybox__ installed. And of course, you need a Stunnel service you can connect to.
You will need either __adb__ or ssh server or a terminal app to run commands. In my case, I am using Terminal Emulator with a Bluetooth keyboard (quite a rookie's way).


Install Stunnel on your device
------------------------------

Download the prebuilt Stunnel for Android [stunnel-4.56-android.zip]( http://www.stunnel.org/downloads.html) from the official Stunnel site. Extract it to a directory, say `/sdcard/stunnel-4.56`.
Run commands in terminal: 

    su -
    mount -o rw,remount /system
    cp -r /sdcard/stunnel-4.56 /system/bin/
    chmod -R 775 /system/bin/stunnel-4.56
    ln -s /system/bin/stunnel-4.56/stunnel /system/bin/stunnel
    ln -s /system/bin/stunnel-4.56/openssl /system/bin/openssl
    mkdir /data/data/org.stunnel

And also prepare `stunnel.conf` and all cert files needed. Put them under, say `/sdcard/stunnel-conf`.
A very simple `stunnel.conf` example:

    client = yes
    delay = no
    fips = no
    compression = zlib
    pid = /data/data/org.stunnel/stunnel.pid
    [http-proxy]
    sslVersion = TLSv1
    accept = 127.0.0.1:10080
    connect = example.com:20080

A bit more complex one:

    client = yes
    delay = no
    fips = no
    compression = zlib
    pid = /data/data/org.stunnel/stunnel.pid
    [another-http-proxy]
    verify = 2
    cert = /etc/certs/client.pem
    key = /etc/certs/client.pem
    CAfile = /etc/certs/cacert.pem 
    accept = 127.0.0.1:10090
    connect = example.com:30080

For debugging, you may need to add these options to the config file:

    debug = 7
    foreground = yes

Copy the config file to `/etc/stunnel.conf`:

    cp /sdcard/stunnel-conf/stunel.conf /etc/stunnel.conf

Copy all certs there, if any. Make sure in `stunnel.conf` options like `cert`, `key`, `CAfile` are using absolute paths to refer cert files. In my case:
    
    cp -r /sdcard/stunnel-conf/certs /etc/
    

Install Stunnel init.d script
-----------------------------

If `/etc/init.d` does not exist in your device, you may need to install [Uni-init](http://forum.xda-developers.com/showthread.php?t=1933849). After installation, open the app, activate and verify.

Prepare an `init.d` script file, `99stunnel`:

    #!/system/bin/sh
    
    STUNNEL=/system/bin/stunnel
    STUNNEL_CONF=/etc/stunnel.conf
    LOG=/data/stunnel.log
    
    if [ -e $LOG ]; then
    	rm $LOG;
    fi; 
    
    if [ ! -e $STUNNEL ]; then
    	echo stunnel binary not found | tee -a $LOG
    	exit 1;
    fi;
    
    if [ ! -e $STUNNEL ]; then
    	echo stunnel configuration file not found | tee -a $LOG
    	exit 1;
    fi;
    
    echo "$( date +"%Y.%m.%d %H:%M:%S" ) starting stunnel" | tee -a $LOG
    $STUNNEL $STUNNEL_CONF | tee -a $LOG
    
    PID="$( pidof stunnel)"
    if [ -z $PID ]; then
    	echo stunnel was not started properly | tee -a $LOG
    	exit 1;
    else
    	echo stunnel running with pid $PID | tee -a $LOG;
    fi;
    
    exit 0;

Copy it to `/etc/init.d/99stunnel`. Make sure that `99stunnel` has the same permissions as other `init.d` scripts there.
Check if the script works:

    /etc/init.d/99stunnel start

Use `netstat -anp` to check if the ports are `LISTEN`. And also remount `/system` as read only:

    mount -o ro,remount /system


References
----------

1. <https://vpn.tv/faq/android-on-vpn-tv-requires-rooted-phone/>
2. <http://forum.xda-developers.com/showthread.php?t=1933849>
3. <http://www.stunnel.org/static/stunnel.html>