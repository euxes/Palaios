Running Shrpx on Ubuntu 12.04 LTS
=================================

[`shrpx`](https://github.com/tatsuhiro-t/spdylay) is a lightweight forward/reverse proxy for SPDY/HTTPS. Among its few features is client certificate verification, which makes `shrpx` a first choice for a scenario that the SPDY proxy is privately shared to a small number of people, and both SPDY proxy and back-end HTTP proxy are running on a budget VPS, and the poor back-end, say `tinyproxy`, supports no user authentication.

The notes below details the configuration of `shrpx` with client certificate verification options enabled.

I. Building `shrpx` from git
----------------------------

Do `apt-get` before building, these packages are required.

    apt-get update && apt-get install build-essential git autoconf automake autotools-dev libtool \
        pkg-config zlib1g-dev libcunit1-dev libssl-dev libxml2-dev libevent-dev

Clone the code into a directory

    git clone https://github.com/tatsuhiro-t/spdylay

Now run

    autoreconf -i
    automake
    autoconf
    ./configure
    make

and 

    make install

It seems that the install script does not work properly on Ubuntu. Run

    ldd `which shrpx`

and you will probably get information like this:

    linux-gate.so.1 =>  (0xb77a6000)
    libssl.so.1.0.0 => /lib/i386-linux-gnu/libssl.so.1.0.0 (0xb7747000)
    libcrypto.so.1.0.0 => /lib/i386-linux-gnu/libcrypto.so.1.0.0 (0xb759c000)
    libevent_openssl-2.0.so.5 => /usr/lib/libevent_openssl-2.0.so.5 (0xb7594000)
    libevent-2.0.so.5 => /usr/lib/libevent-2.0.so.5 (0xb754e000)
    libspdylay.so.7 => not found
    libstdc++.so.6 => /usr/lib/i386-linux-gnu/libstdc++.so.6 (0xb7454000)
    libgcc_s.so.1 => /lib/i386-linux-gnu/libgcc_s.so.1 (0xb7436000)
    libpthread.so.0 => /lib/i386-linux-gnu/libpthread.so.0 (0xb741a000)
    libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xb7270000)
    libdl.so.2 => /lib/i386-linux-gnu/libdl.so.2 (0xb726b000)
    libz.so.1 => /lib/i386-linux-gnu/libz.so.1 (0xb7255000)
    libevent_core-2.0.so.5 => /usr/lib/libevent_core-2.0.so.5 (0xb722b000)
    librt.so.1 => /lib/i386-linux-gnu/librt.so.1 (0xb7221000)
    libm.so.6 => /lib/i386-linux-gnu/libm.so.6 (0xb71f5000)
    /lib/ld-linux.so.2 (0xb77a7000)

Run `locate libspdylay.so` or `find /usr/local -name libspdylay`, you will *probably* get:

    /usr/local/lib/libspdylay.so
    /usr/local/lib/libspdylay.so.7
    /usr/local/lib/libspdylay.so.7.0.2

So these library files (actually only one library file, other two are symbolic links) were being copied to a directoy `shrpx` cannot find. Simply making a symbolic link will solve the problem:

    ln -s /usr/local/lib/libspdylay.so /lib/i386-linux-gnu/libspdylay.so.7

II. OpenSSL prerequisite 
------------------------

Before configuring `shrpx`, there are some SSL certs that should be prepared, including a root CA cert, a pair of server-side cert and key, some pairs of client-side cert and key. And of course you can use a self-signed server-side cert.

This part is almost the same as Part II of [Setting up a Stunnel-secured Proxy Server](https://github.com/vvord/kalos/blob/master/setting_up_stunnel%2Bproxy.md). So please refer to that for more details. 

Assume you have already read the previously mentioned part, and successfully created your root CA `cacert.crt`. Now generate server-side key and cert:

    openssl req -config ./openssl.cnf -new -keyout server.key \
        -out server_req.pem -days 3650 -nodes
    openssl ca -config ./openssl.cnf -policy policy_anything \
        -out server_signed.pem -infiles server_req.pem
    openssl x509 -in server_signed.pem -out server.crt

and client-side key and cert:

    openssl req -config ./openssl.cnf -new -keyout shrpx_client.key \
        -out shrpx_client_req.pem -days 3650 -nodes
    openssl ca -config ./openssl.cnf -policy policy_anything \
        -out shrpx_client_signed.pem -infiles shrpx_client_req.pem
    openssl x509 -in shrpx_client_signed.pem -out shrpx_client.crt

Concatenate certs:

    cat shrpx_client.key shrpx_client.crt > shrpx_client.pem
    cat cacert.crt shrpx_client.crt > trusted_certs.crt

And because Windows doesn't accept PEM encoded private key, for Windows use you have to convert the client cert to `.pfx` format:

    openssl pkcs12 -export -in shrpx_client.pem -out shrpx_client.pfx

III. Configuring `shrpx`
------------------------

Assume you already have an HTTP proxy running on your local machine, with its listening port 8888. It will be used as a back-end proxy. 

Copy all certs to `/etc/shrpx`:

    mkdir /etc/shrpx
    mkdir /etc/shrpx/certs
    cp server.crt server.key trusted_certs.crt /etc/shrpx/certs

Create `shrpx.conf` under `/etc/shrpx` as following:

    #binding address, listening port
    frontend=0.0.0.0,18443
    #back-end proxy address
    backend=127.0.0.1,8888
    #relative path seems not working in conf file.
    private-key-file=/etc/shrpx/certs/server.key
    certificate-file=/etc/shrpx/certs/server.crt
    verify-client=yes
    verify-client-cacert=/etc/shrpx/certs/trusted_certs.crt
    #enable spdy proxy mode
    spdy-proxy=yes
    daemon=yes
    accesslog=yes
    workers=1

For details please refer to `shrpx.conf.sample`. 

An init-script is not provided along with the source code, so to run `shrpx` on start up, either you add `shrpx --conf path/to/conf/file` to `crontab` or use an init-script modified from `/etc/init.d/skeleton`. Following is an example of init-script for `shrpx` modified from `skeleton`:

    ! /bin/sh
    ### BEGIN INIT INFO
    # Provides:          shrpx
    # Required-Start:    $network $remote_fs $syslog
    # Required-Stop:     $remote_fs $syslog
    # Default-Start:     2 3 4 5
    # Default-Stop:      0 1 6
    # Short-Description: shrpx initscript
    # Description:       Shrpx - A SPDY/HTTPS proxy server.
    ### END INIT INFO

    # Do NOT "set -e"

    # PATH should only include /usr/* if it runs after the mountnfs.sh script
    PATH=/usr/local/sbin:/sbin:/usr/sbin:/bin:/usr/bin
    DESC="shrpx"
    NAME=shrpx
    #DAEMON=/usr/sbin/$NAME
    DAEMON=/usr/local/bin/$NAME
    DAEMON_ARGS="--conf /etc/shrpx/shrpx.conf"
    #DAEMON_ARGS="--conf /root/shrpx.conf"
    PIDFILE=/var/run/$NAME.pid
    SCRIPTNAME=/etc/init.d/$NAME

    # Exit if the package is not installed
    [ -x "$DAEMON" ] || exit 0

    # Read configuration variable file if it is present
    [ -r /etc/default/$NAME ] && . /etc/default/$NAME

    # Load the VERBOSE setting and other rcS variables
    . /lib/init/vars.sh

    # Define LSB log_* functions.
    # Depend on lsb-base (>= 3.2-14) to ensure that this file is present
    # and status_of_proc is working.
    . /lib/lsb/init-functions

    #
    # Function that stops the daemon/service
    #
    do_stop()
    {
        # Return
        #   0 if daemon has been stopped
        #   1 if daemon was already stopped
        #   2 if daemon could not be stopped
        #   other if a failure occurred
        start-stop-daemon --stop --quiet --retry=TERM/30/KILL/5 --pidfile $PIDFILE --name $NAME
        RETVAL="$?"
        [ "$RETVAL" = 2 ] && return 2
        # Wait for children to finish too if this is a daemon that forks
        # and if the daemon is only ever run from this initscript.
        # If the above conditions are not satisfied then add some other code
        # that waits for the process to drop all resources that could be
        # needed by services started subsequently.  A last resort is to
        # sleep for some time.
        start-stop-daemon --stop --quiet --oknodo --retry=0/30/KILL/5 --exec $DAEMON
        [ "$?" = 2 ] && return 2
        # Many daemons don't delete their pidfiles when they exit.
        rm -f $PIDFILE
        return "$RETVAL"
    }

    #
    # Function that sends a SIGHUP to the daemon/service
    #
    do_reload() {
        #
        # If the daemon can reload its configuration without
        # restarting (for example, when it is sent a SIGHUP),
        # then implement that here.
        #
        start-stop-daemon --stop --signal 1 --quiet --pidfile $PIDFILE --name $NAME
        return 0
    }

    case "$1" in
      start)
        [ "$VERBOSE" != no ] && log_daemon_msg "Starting $DESC" "$NAME"
        echo -n "Starting $DESC"
        do_start
        case "$?" in
            0|1) [ "$VERBOSE" != no ] && log_end_msg 0 ;;
            2) [ "$VERBOSE" != no ] && log_end_msg 1 ;;                                       esac
        ;;
      stop)
        [ "$VERBOSE" != no ] && log_daemon_msg "Stopping $DESC" "$NAME"
        echo -n "Stopping $DESC"
        do_stop
        case "$?" in
            0|1) [ "$VERBOSE" != no ] && log_end_msg 0 ;;
            2) [ "$VERBOSE" != no ] && log_end_msg 1 ;;
        esac
        ;;
      status)
           status_of_proc "$DAEMON" "$NAME" && exit 0 || exit $?
           ;;
      #reload|force-reload)
        #
        # If do_reload() is not implemented then leave this commented out
        # and leave 'force-reload' as an alias for 'restart'.
        #
        #log_daemon_msg "Reloading $DESC" "$NAME"
        #do_reload
        #log_end_msg $?
        #;;
      restart|force-reload)
        #
        # If the "reload" option is implemented then remove the
        # 'force-reload' alias
        #
        log_daemon_msg "Restarting $DESC" "$NAME"
        echo -n "Restarting $DESC"
        do_stop
        case "$?" in
          0|1)
            do_start
            case "$?" in
                0) log_end_msg 0 ;;
                1) log_end_msg 1 ;; # Old process is still running
                *) log_end_msg 1 ;; # Failed to start
            esac
            ;;
          *)
            # Failed to stop
            log_end_msg 1
            ;;
        esac
        ;;
      *)
        #echo "Usage: $SCRIPTNAME {start|stop|restart|reload|force-reload}" >&2
        echo "Usage: $SCRIPTNAME {start|stop|status|restart|force-reload}" >&2
        exit 3
        ;;
    esac

    :

Please note that the modified script has not been fully tested, and is buggy. Copy the script to `/etc/init.d/`, naming it `shrpx`, and do

    chmod a+x /etc/init.d/shrpx
    update-rc.d shrpx defaults
    /etc/init.d/shrpx start

Tyoe `ps ax | grep shrpx` and `netstat -anp | grep 18443` to check if `shrpx` is working.


IV. Configuring client browser
------------------------------

Since the SPDY proxy you are going to connect requests client certificate verification, you will not be able to use it if the certificate is not properly imported.

And so far the mere browsers that supports SPDY proxy is Chrome/Chromium. The SPDY proxy will only serve as an HTTPS proxy if it is used in other browsers.

__a. Import Client and CA Certificate on Windows__

Since Windows doesn't accept PEM encoded private key, choose the `.pfx` one. It is recommended that you use `certmgr.msc` to import the certificate.

STEPS:

1) Launch `certmgr.msc`;

2) Right click on "Personal" folder, choose "All task \ Import ...";

3) Choose the `.pfx` file you want to import (if it was not listed, select "All files" in Open dialog);

4) Type the password. 

5) [optional] Click "Refresh" button on the certmgr window.

The certificate will be in "Personal\Certificates" folder if it is correctly imported.

And double click `cacert.crt` to install the CA cert to "Trusted Root Certification Authorities".


__b. Import PAC file to Chrome__

A PAC file for SPDY proxy is no more than a common PAC file, except for its returning HTTPS address instead of SOCKS or HTTP ones:

    function FindProxyForURL(url, host) {
        return "HTTPS spdy.proxy.address:portnumber";
    }

You may directly use the PAC file with Chrome like this:

    /path/to/chrome.exe --proxy-pac-url=file:///path/to/config.pac --use-npn

Or if you are also using SwitchySharp extension with Chrome, you can import the PAC file to the extension, in which the file will be stored as a base64 encoded string.

Please note that if you fill the SPDY proxy address to "HTTPS Proxy" blanket in SwitchySharp, the SPDY proxy will only act as an HTTPS proxy, not a SPDY one. The only way for now to use SPDY proxy with SwitchySharp is through PAC file.

STEPS:

1) Create a new profile in SwitchSharp;

2) Under "Profile Details", Choose "Automatic Configuration", and click "Import PAC File";

3) Click "Save".

You will be prompted to choose a certificate when connecting to the SPDY proxy.


V. References
-------------

1. <https://github.com/tatsuhiro-t/spdylay>
2. <http://www.chromium.org/spdy/spdy-proxy>
3. <https://www.openssl.org/docs/apps/pkcs12.html>
4. <http://stackoverflow.com/questions/808669/convert-a-cert-pem-certificate-to-a-pfx-certificate>
5. <http://windows.microsoft.com/en-us/windows/import-export-certificates-private-keys#1TC=windows-7>
6. <https://www.npmjs.org/package/spdyproxy>
7. <http://support.microsoft.com/kb/179380>


