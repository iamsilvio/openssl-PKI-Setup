# openssl PKI setup

let's start by clarifying a few terms

* PKI - publik key infrastructure
* Root CA - TODO
* air gapped - the computer is not connected to a network, no WLAN, no LAN,
                no Bluetooth or any other radio link.

## Air gapped Root CA

lets start with a air gapped Root CA

create some folders, change permissions and create some files

``` terminal
mkdir -p $HOME/ca/newcerts $HOME/ca/db $HOME/ca/private
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