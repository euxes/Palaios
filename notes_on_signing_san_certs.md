Notes on Signing Subject Alternative Names (SANs) Certificates
===============================================================

I. Creating a Root CA
---------------------

Skip this part if you are to issue self-signed certificates using `openssl x509` or you have already got a root CA. Otherwise just refer to the __OpenSSL prerequisite__ part of my notes _Setting up a Stunnel-secured Proxy Server_.

To be brief, run 

    mkdir ~/ssl_certs
    cp /usr/lib/ssl/openssl.cnf ~/ssl_certs/

Edit `~/ssl_certs/openssl.cnf`

    [ CA_default ]
    dir             = .                     # Where everything is kept
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

Run

    cd ~/ssl_certs
    touch index.txt
    echo "01" > serial
    mkdir certs crl newcerts private

and

    openssl req -config ./openssl.cnf -new -x509 -keyout private/cakey.pem \
        -out cacert.pem -days 3650
    

II. Creating a Certificate Signing Request (CSR) with SANs
----------------------------------------------------------

Well, before creating a CSR with SANS, you probably need copy a new `openssl.cnf`.

    cp openssl.cnf foo.cnf
    
Edit `foo.cnf`, uncomment `x509_extensions` and `req_extensions` under `[req]` section,

    [req]
    x509_extensions = v3_req
    req_extensions = v3_req
    distinguished_name = req_distinguished_name

and modify `[v3_req]` section, and append `[alt_names]` section.

    [v3_req]
    subjectAltName = @alt_names
    basicConstraints = CA:FALSE
    keyUsage = nonRepudiation, digitalSignature, keyEncipherment

    [alt_names]
    DNS.1 = foobar.example.com
    DNS.2 = foo.example.com
    DNS.3 = f.example.com

Now, create a CSR.

    openssl req -config ./foo.cnf -new -keyout foo_server_key.pem \
        -out foo_server_req.pem -days 3650 -nodes

Verify the CSR using 

    openssl req -text -noout -verify -in foo_server_req.pem
    
If created correctly, there should be `X509v3 Subject Alternative Name` section in output.


III. Signing the CSR
--------------------

Sign the CSR using

    openssl ca -config ./foo.cnf -policy policy_anything -out foo_server_signed.pem \
        -extensions v3_req -infiles foo_server_req.pem 

Or you can create another `.cnf` file (say, `foo_req.cnf`) that only contains following lines, in the scenario that you are signing the CSR from others. 

    [v3_req]
    subjectAltName = @alt_names
    basicConstraints = CA:FALSE
    keyUsage = nonRepudiation, digitalSignature, keyEncipherment

    [alt_names]
    DNS.1 = foobar.example.com
    DNS.2 = foo.example.com
    DNS.3 = f.example.com

and use `foo_req.cnf` along with `-extfile` option. 

    openssl ca -config ./openssl.cnf -policy policy_anything -out foo_server_signed.pem \
        -extfile foo_req.cnf -extensions v3_req -infiles foo_server_req.pem  

Note that in both ways, the `-infiles` option should be the last option, as the [`openssl ca` doc][1] says:

> __-infiles__   
>    if present this should be the last option, all subsequent arguments are assumed to the the[sic] names of files containing certificate requests.

Errors such as 

> ...  
> openssl "-extensions": No such file or directory   
> ...   

will occur if putting `-infiles` option in the middle.

Verify the signed certificate.

    openssl x509 -text -noout -in foo_server_signed.pem


IV. References
--------------

1. <https://www.openssl.org/docs/apps/ca.html>
2. <http://blog.zencoffee.org/2013/04/creating-and-signing-an-ssl-cert-with-alternative-names/>

[1]: https://www.openssl.org/docs/apps/ca.html






