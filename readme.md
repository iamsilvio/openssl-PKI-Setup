# openssl PKI setup

let's start by clarifying a few terms

* PKI - public key infrastructure
* Root CA - TODO
* air gapped - the computer is not connected to a network, no WLAN, no LAN,
                no Bluetooth or any other radio link.

## Air gapped Root Certificate Authority

Add the path to the environment

``` shell
export ROOTCADIR=$HOME/ca
```

create some folders, change permissions and create some files

``` shell
mkdir -p $ROOTCADIR/{newcerts,db,private,crl}
chmod 0700 $ROOTCADIR/private

touch $ROOTCADIR/db/root-ca.db
touch $ROOTCADIR/db/root-ca.db.attr

echo 01 > $ROOTCADIR/db/root-ca.crt.srl
echo 01 > $ROOTCADIR/db/root-ca.crl.srl
```

next we create the elliptic curve parameters

``` shell
openssl ecparam -out $ROOTCADIR/private/ec-secp384r1.pem -name secp384r1
```

copy the `root-ca.conf` to your `$ROOTCADIR/ca.conf`
  
create the self Signed Root Certificate

``` shell
openssl req -config $ROOTCADIR/ca.conf -new -x509 -days 3650\
            -sha512 -newkey ec:$ROOTCADIR/private/ec-secp384r1.pem \
            -keyout $ROOTCADIR/private/ca.key -out $ROOTCADIR/ca.crt \
            -extensions root_ca_ext
```

you can verify the certificate with

`openssl x509 -in $ROOTCADIR/ca.crt -text`

at the end we create a certificate revocation list

``` shell
openssl ca -gencrl -config $ROOTCADIR/ca.conf -out $ROOTCADIR/crl/root-ca.crl
```

## Signing Certificate Authority

Add the path to the environment

``` shell
export CADIR=$HOME/int-ca
```

create some folders again, change permissions and create some files

``` shell
mkdir -p $CADIR/{newcerts,db,private,crl}
chmod 0700 $CADIR/private

touch $CADIR/db/signing-ca.db
touch $CADIR/db/signing-ca.db.attr

echo 01 > $CADIR/db/signing-ca.crt.srl
echo 01 > $CADIR/db/signing-ca.crl.srl
```

next we create the elliptic curve parameters

``` shell
openssl ecparam -out $CADIR/private/ec-secp384r1.pem -name secp384r1
```

copy the `signing-ca.conf` to your `$CADIR/ca.conf`
  
create the certificate request for the Signing CA

``` shell
openssl req -config $CADIR/ca.conf -new \
            -sha512 -newkey ec:$CADIR/private/ec-secp384r1.pem \
            -keyout $CADIR/private/ca.key \
            -out $CADIR/ca.csr
```

now we get a USB stick and go to our Root CA

``` shell
openssl ca \
    -config $ROOTCADIR/ca.conf  \
    -in $CADIR/ca.csr \
    -out $CADIR/ca.crt \
    -extensions signing_ca_ext
```

at the end we create a certificate revocation list

``` shell
openssl ca -gencrl -config $CADIR/ca.conf -out $CADIR/crl/int-ca.crl
```

## Generating Certificate Signing Requests with elliptic keys

With `openssl ecparam -list_curves` you get a list with all available elliptic curves.
For compatibility reasons we chose a secp384r1.

lets jump to our web server first and generate a key and the Certificate Signing Request

``` shell
openssl ecparam -genkey -name secp384r1 | openssl ec -out lab.key.pem

openssl req -new -subj "/CN=lab.deleteonerror.com" \
                  -addext "subjectAltName = DNS:lab.deleteonerror.com, IP:168.119.55.72" \
                  -key lab.key -out lab.csr
```

now you have to transport the Request to the Signing-CA and execute

``` shell
openssl ca \
    -config $CADIR/ca.conf \
    -in lab.csr\
    -out lab.crt \
    -extensions server_ext
```

Grab your certificate and you're done, at least you're close to what's called done.
You may need to convert your certificate or merge it into a file, depending on where you use the certificate.



``` shell
openssl req -config client_req.conf -new \
            -sha512 -newkey ec:ec-secp384r1.pem \
            -keyout client.key \
            -out client.csr

openssl ca \
    -config $CADIR/ca.conf \
    -in client.csr\
    -out client.crt \
    -extensions client_ext

```




## Revoke Certificates

Only two steps are needed to revoke a certificate.

### Step 1

At the Certificate Authority where the Certificate was Issued you just run

``` shell
openssl ca -config $CADIR/ca.conf -revoke [TheCertificaateYouWantToRevoke.pem]
```

If you don't have the PEM file by hand, you can find it in the `$CADIR/newcerts` folder.

### Step 2

after you have revoked the certificate you have to publish a new CRL by executing

``` shell
openssl ca -gencrl -config $CADIR/ca.conf -out $CADIR/crl/int-ca.crl
```
