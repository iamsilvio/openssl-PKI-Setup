# openssl PKI setup

let's start by clarifying a few terms

* PKI - publik key infrastructure
* Root CA - TODO
* air gapped - the computer is not connected to a network, no WLAN, no LAN,
                no Bluetooth or any other radio link.

## Air gapped Root Certificate Authority

create some folders, change permissions and create some files

``` terminal
mkdir -p $HOME/ca/newcerts $HOME/ca/db $HOME/ca/private $HOME/ca/crl
chmod 700 $HOME/ca/private

touch $HOME/ca/db/root-ca.db
touch $HOME/ca/db/root-ca.db.attr

echo 01 > $HOME/ca/db/root-ca.crt.srl
echo 01 > $HOME/ca/db/root-ca.crl.srl
```

Add the path to the environment

``` terminal
export CADIR=$HOME/ca

```

next we create the elliptic curve parameters

``` terminal
openssl ecparam -out $CADIR/private/ec-secp384r1.pem -name secp384r1

```

create the self Signed Root Certificate

``` terminal
openssl req -config $CADIR/root-ca.conf -new -x509 \
            -days 3652 -sha512 -newkey ec:$CADIR/private/ec-secp384r1.pem \
            -keyout $CADIR/private/root-ca.key -out $CADIR/root-ca.crt \
            -extensions root_ca_ext
```

you can verify the certificate with

`openssl x509 -in $CADIR/root-ca.crt -text`

at the end we create a certificate revocation list

``` terminal
openssl ca -gencrl \
    -config $CADIR/root-ca.conf \
    -out $CADIR/crl/root-ca.crl
```

## Signing Certificate Authority

create some folders again, change permissions and create some files

``` terminal
mkdir -p $HOME/ca/newcerts $HOME/ca/db $HOME/ca/private $HOME/ca/crl
chmod 700 $HOME/ca/private

touch $HOME/ca/db/signing-ca.db
touch $HOME/ca/db/signing-ca.db.attr

echo 01 > $HOME/ca/db/signing-ca.crt.srl
echo 01 > $HOME/ca/db/signing-ca.crl.srl
```

Add the path to the environment

``` terminal
export CADIR=$HOME/ca

```

next we create the elliptic curve parameters

``` terminal
openssl ecparam -out $CADIR/private/ec-secp384r1.pem -name secp384r1

```

create the certificate request for the Signing CA

``` terminal
openssl req -config $CADIR/signing-ca.conf -new \
            -days 731 -sha512 -newkey ec:$CADIR/private/ec-secp384r1.pem \
            -keyout $CADIR/private/signing-ca.key \
            -out $CADIR/signing-ca.csr
```

now we get a USB stick and go to our Root CA

``` terminal
openssl ca \
    -config $CADIR/root-ca.conf  \
    -in $CADIR/signing-ca.csr \
    -out $CADIR/signing-ca.crt \
    -extensions signing_ca_ext
```

at the end we create a certificate revocation list

``` terminal
openssl ca -gencrl \
    -config $CADIR/signing-ca.conf \
    -out $CADIR/crl/signing-ca.crl
```

## Generating Certificate Signing Requests with elliptic keys

With `openssl ecparam -list_curves` you get a list with all available elliptic curves.
For compatibility reasons we chose a secp384r1.

lets jump to our web server first and generate a key and the Certificate Signing Request

``` terminal
openssl ecparam -genkey -name secp384r1 | openssl ec -out /etc/ssl/nginx/pki.deleteonerror.com.key

openssl req -new -sha512 \
            -key /etc/ssl/nginx/pki.deleteonerror.com.key \
            -out /opt/letsencrypt/pki.deleteonerror.com.csr \
            -subj "/CN=pki.deleteonerror.com"
```

or if you need a SAN certificate

``` terminal
openssl req -new -sha512 \
            -key /etc/ssl/nginx/pki.deleteonerror.com.key \
            -subj "/" -reqexts SAN \
            -config <(cat /etc/ssl/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:pki.deleteonerror.com DNS:crl.deleteonerror.com")) \
            > /opt/letsencrypt/pki.deleteonerror.com.csr
```

now you have to transport the Request to the Signing-CA and execute

```terminal
openssl ca \
    -config $CADIR/signing-ca.conf \
    -in $CADIR/newcerts/pki.deleteonerror.com.csr\
    -out $CADIR/newcerts/pki.deleteonerror.com.cst \
    -extensions server_ext
```

Grab your certificate and you're done, at least you're close to what's called done.
You may need to convert your certificate or merge it into a file, depending on where you use the certificate.
