Running Shadowsocks-libev on Ubuntu
===================================

I. Installation
---------------
Make sure you have `git` and packages for compilation installed, if no, `apt-get install` them first:

    sudo apt-get install git
    sudo apt-get install build-essential autoconf libtool libssl-dev

Fetch the latest source code from github, and install after compilation:

    git clone https://github.com/madeye/shadowsocks-libev
    cd shadowsocks-libev
    ./configure && make
    sudo make install

And copy the config file and the init.d script to sub-directories under `/etc/` respectively:

    cd debian
    mkdir /etc/shadowsocks
    cp config.json /etc/shadowsocks
    cp shadowsocks.init /etc/init.d/shadowsocks
    chmod a+x /etc/init.d/shadowsocks

II. Configuration
-----------------
Shadowsocks-libev won't be running properly with default `config.json` and `init.d` script. You have to do some modifications:

    vi /etc/shadowsocks/config.json

Following is an example of working `config.json`:

    {
        "server":"0.0.0.0",
        "server_port":17777,
        "local_port":7777,
        "password":"SuperCowPower",
        "timeout":600,
        "method":"aes-256-cfb"
    }

`server` is the ip address that Shadowsocks-libev will be listening to , `0.0.0.0` will just be fine. `server_port` is the port that your Shadowsocks client will connect to.

Locate `ss-server`: 

    which ss-server
    
You will get something like: 

    /usr/local/bin/ss-server

Then change `DAEMON` value to this _path/to/file_ in `/etc/init.d/shadowsocks`:

    DAEMON=/usr/local/bin/ss-server
    
And also comment out `[ -r /etc/default/$NAME ] && . /etc/default/$NAME`:

    # Read configuration variable file if it is present
    #[ -r /etc/default/$NAME ] && . /etc/default/$NAME

And insert following lines _directly_ after:

    # Enable during startup?
    START=yes
    
    # Configuration file
    CONFFILE="/etc/shadowsocks/config.json"

Now install the `init.d` script:

    update-rc.d shadowsocks defaults
    
And start the service:
    
    sudo /etc/init.d/shadowsocks start
    
Use `netstat -anp | grep 17777` to check if it is _listening_.

III. Misc.
----------
Run `ss-server` in foreground with a configuration file:

    ss-server -c /etc/shadowsocks/config.json
    
In background, even when you disconnect from terminal, an alternative to `screen`:

    nohup ss-server -c /etc/shadowsocks/config.json &


IV. References
--------------
1. <https://github.com/madeye/shadowsocks-libev/blob/master/README.md>

