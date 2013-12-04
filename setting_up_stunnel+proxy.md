Setting up a Stunnel-secured Proxy Server
=========================================

Assume hereafter you are using the latest version of Ubuntu Server, and are using root account.

I. Setting up a proxy server
----------------------------

First you need a proxy server, such as __Tinyproxy__, __Dante__, running on your VPS. In the case of Tinyproxy, install it using

    apt-get install tinyproxy

and configure it using

    vi /etc/tinyproxy.conf

Actually even you did not configure it, Tinyproxy will still be running properly. Here is an example of `tinyproxy.conf`:

    User nobody
    Group nogroup
    Port 8888
    Timeout 600
    DefaultErrorFile "/usr/share/tinyproxy/default.html"
    StatFile "/usr/share/tinyproxy/stats.html"
    Logfile "/var/log/tinyproxy/tinyproxy.log"
    LogLevel Info
    PidFile "/var/run/tinyproxy/tinyproxy.pid"
    MaxClients 25
    MinSpareServers 5
    MaxSpareServers 20
    StartServers 10
    MaxRequestsPerChild 0
    Allow 127.0.0.1
    ViaProxyName "tinyproxy"
    ConnectPort 443
    ConnectPort 563

Start it using 

    /etc/init.d/tinyproxy restart

Make sure the port 8888 is open, using

    netstat -anp | grep 888

or 

    nmap localhost -p8888

If the service is up, the port 8888 should be in the status of `LISTEN` or `open`.

II. OpenSSL prerequisite 
------------------------

Assume you will generate and sign certificates under directory, say, `~/ssl_certs/`.

Copy the openssl.cnf to ~/ssl_certs/, using 

    mkdir ~/ssl_certs
    cp /usr/lib/ssl/openssl.cnf ~/ssl_certs/

and configure it to minimize some typing work, using

    vi ~/ssl_certs/openssl.cnf

under section `[ CA_default ]` :

    [ CA_default ]
    dir             = ./            # Where everything is kept
    certs           = $dir/certs            # Where the issued certs are kept
    crl_dir         = $dir/crl              # Where the issued crl are kept
    database        = $dir/index.txt        # database index file.
    #unique_subject = no                    # Set to 'no' to allow creation of
                                    # several ctificates with same subject.
    new_certs_dir   = $dir/newcerts         # default place for new certs.
    certificate     = $dir/cacert.pem       # The CA certificate
    serial          = $dir/serial           # The current serial number
    crlnumber       = $dir/crlnumber        # the current crl number
                                    # must be commented out to leave a V1 CRL
    crl             = $dir/crl.pem          # The current CRL
    private_key     = $dir/private/cakey.pem# The private key
    RANDFILE        = $dir/private/.rand    # private random number file

and

    default_days    = 3650                   # how long to certify for

You can also modify settings under section `[ req_distinguished_name ]`.

Now enter directory `~/ssl_cersts`, and create an empty file `index.txt`, a file `serial` containing only `01`, and also create sub-directories mentioned above:

    cd ~/ssl_certs
    touch index.txt
    echo "01" > serial
    mkdir certs crl newcerts private
    
Now create a root CA certificate valid for 10 years:

    openssl req -config ./openssl.cnf -new -x509 -keyout private/cakey.pem \
        -out cacert.pem -days 3650
        
You will be prompted to enter a PEM passphrase, which will be used later.

Now generate and sign a certificate request for server-side use. First, generate a request:

    openssl req -config ./openssl.cnf -new -keyout server_key.pem \
        -out server_req.pem -days 3650 -nodes
      
It also creates a server-side private key “server_key.pem”. Sign the request:

    openssl ca -config ./openssl.cnf -policy policy_anything \
        -out server_signed.pem -infiles server_req.pem
        
You will need to enter the passphrase previously mentioned. If succeeded, it will return

    Write out database with 1 new entries
    Data Base Updated

And the file `serial` will be updated to `02`. And run 

    openssl x509 -in server_signed.pem -out server_cert.pem

to rip off extra infomation other than the encoded certificate. Concatenate `server_key.pem` and `server_cert.pem` into one file:

    cat server_key.pem server_cert.pem > server.pem
    
For client-side Stunnel, it is almost the same:

    openssl req -config ./openssl.cnf -new -keyout client_key.pem \
        -out client_req.pem -days 3650 -nodes
    openssl ca -config ./openssl.cnf -policy policy_anything \
        -out client_signed.pem -infiles client_req.pem
    openssl x509 -in client_signed.pem -out client_cert.pem
    cat client_key.pem client_cert.pem > client.pem
    
Now for the trusted certificates part. Stunnel supports two ways of storing your trusted certificates: one is single file with multiple trusted certificates, the other each certificates in its own file. We choose the former, which is relatively easier. The file will be of the form:

    -----BEGIN CERTIFICATE-----
    certificate #1 data here
    -----END CERTIFICATE-----
    -----BEGIN CERTIFICATE-----
    certificate #2 data here
    -----END CERTIFICATE-----

Concatenate the root CA certificate, all client certificates you have generated (ensure that you have not included any private keys). The root CA certificate goes first:

    cat cacert.pem client_cert.pem > trusted_certs.pem


III. Stunnel configuration
--------------------------

Install stunnel in a most easy way:

    apt-get install stunnel
    
The problem is that Stunnel in repository is not always the latest. If you want to use some new features, you should always make it yourself.

Edit the file `/etc/default/stunnel4`:

    vi /etc/default/stunnel4 

Change the setting `ENABLED=0` to `ENABLED=1`. Copy `stunnel.conf` to `/etc/stunnel`:

    cp /usr/share/doc/stunnel4/examples/stunnel.conf-sample \
        /etc/stunnel/stunnel.conf
        
And also copy `server.pem`, `trusted_certs.pem` to `/etc/stunnel/`:

    cp ~/ssl_certs/server.pem ~/ssl_certs/trusted_certs.pem /etc/stunnel/
    
Edit `stunnel.conf`, change the settings accordingly, comment out unnecessary ones. Here is a tidy example of `stunnel.conf`:

    pid = /stunnel4.pid
    cert = /etc/stunnel/server.pem
    key = /etc/stunnel/server.pem
    verify = 3
    CAfile = /etc/stunnel/trusted_certs.pem
    options = NO_SSLv2
    
    [tinyproxy-http]
    client = no
    accept = 18888
    connect = 8888
    delay = no
    
Start the service:

    /etc/init.d/stunnel4 start

If configured correctly, the port 18888 should be _listening_.

    netstat -anp | grep 18888

should return something like this:

    tcp	0	0 0.0.0.0:18888	0.0.0.0:*	LISTEN		12621/stunnel4

Now the work for server-side part is done. 

For the client-side, assume you are using Stunnel on Windows. Copy `cacert.pem` and `client.pem` to the place where `stunnel.exe` is located. Also edit `stunnel.conf`:

    verify = 2
    client = yes
    compression = zlib
    cert = client.pem
    key = client.pem
    CAfile = cacert.pem 
    
    [tinyproxy-http]
    accept = 127.0.0.1:10080
    connect = example.com:18888
    delay = no
    
The option `connect` should be the address (IP address, or a domain name) of the machine where runs Stunnel, ending with `:port_number`.

Voilà. Now the local port 10080 is for you to connect. 

IV. References
--------------

1. <https://www.stunnel.org/static/stunnel.html>
2. <https://www.stunnel.org/faq.html>
3. <https://www.stunnel.org/howto.html>
4. <http://www.tldp.org/HOWTO/SSL-Certificates-HOWTO/>
5. <http://www.openssl.org/docs/apps/openssl.html>
