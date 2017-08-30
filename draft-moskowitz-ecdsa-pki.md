---
stand_alone: true
ipr: trust200902
docname: draft-moskowitz-ecdsa-pki-latest
cat: info
pi:
  toc: 'yes'
  symrefs: 'yes'
  sortrefs: 'yes'
  compact: 'yes'
  subcompact: 'no'
  iprnotified: 'no'
  strict: 'no'
title: Guide for building an ECC pki
abbrev: PKI Guide
area: Security Area
wg: wg TBD
kw: RFC
date: 2017
author:
- ins: R. Moskowitz
  name: Robert Moskowitz
  org: Huawei
  street: ''
  city: Oak Park
  region: MI
  code: '48237'
  email: rgm@labs.htt-consult.com
- ins: H. Birkholz
  name: Henk Birkholz
  org: Fraunhofer SIT
  street: Rheinstrasse 75
  city: Darmstadt
  code: '64295'
  country: Germany
  email: henk.birkholz@sit.fraunhofer.de
- ins: L. Xia
  name: Liang Xia
  org: Huawei
  street: No. 101, Software Avenue, Yuhuatai District
  city: Nanjing
  country: China
  email: Frank.xialiang@huawei.com
ref: {}
normative:
  RFC2119:
informative:
#  IEEE.802.1AR_2009:
  RFC2315:
  RFC2818:
  RFC7292:

--- abstract


This memo provides a guide for building a PKI (Public Key
Infrastructure) using openSSL. All certificates in this guide are
ECDSA, P-256, with SHA256 certificates. Along with common End
Entity certificates, this guide provides instructions for creating
IEEE 802.1AR iDevID Secure
Device certificates.

--- middle

# Introduction {#intro}

The IETF has a plethora of security solutions targeted at IoT.
Yet all too many IoT products are deployed with no or
improperly configured security. In particular resource
constrained IoT devices and non-IP IoT networks have not been
well served in the IETF.

Additionally, more IETF (e.g. DOTS, NETCONF) efforts are
requiring secure identities, but are vague on the nature of
these identities other than to recommend use of X.509 digital
certificates and perhaps TLS.

This effort provides the steps, using the openSSL application,
to create such a PKI of ECDSA certificates. The goal is that
any developer or tester can follow these steps, create the
basic objects needed and establish the validity of the
standard/program design. This guide can even be used to create
a production PKi, though additional steps need to be taken.
This could be very useful to a small vendor needing to include
802.1AR iDevIDs in their product.

This guide was tested with openSSL 1.1.0f on Fedora 26 and
creates PEM-based certificates. DER based certificates fails
(see {{DER}}). Also, at this time, CRL and OCSP
support is for future work.


# Terms and Definitions {#terms}

## Requirements Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described
in [RFC 2119](#RFC2119).


## Notations

This section will contain notations


## Definitions

TBD



# The Basic PKI feature set {#BasicPKI}

A basic pki has two levels of hierarchy: Root and Intermediate. The
Root level has the greatest risk, and is the least used. It only
signs the Intermediate level signing certificate. As such, once
the Root level is created and signs the Intermediate level
certificate it can be locked up. In fact, the Root level could
exist completely on a mSD boot card for an ARM small computer like
a RaspberryPi. A copy of this card came be made and securely
stored in a different location.

The Root level contains the Root certificate private key, a
database of all signed certificates, and  the public certificate.
It can also contain the Intermediate level public certificate and a
Root level CRL.

The Intermediate level contains the Intermediate certificate
private key, the public certificate, a database of all signed
certificates, the certificate trust chain, and Intermediate level
CRL. It can also contain the End Entity public certificates. The
private key file needs to be keep securely. For example as with
the Root level, a mSD image for an ARM computer could contain the
complete Intermediate level. This image is kept offline. The End
Entity CSR is copied to it, signed, and then the signed certificate
and updated database are moved to the public image that lacks the
private key.

For a simple test pki, all files can be kept on a single system
that is managed by the tester.

End Entities create a key pair and a Certificate Signing Request
(CSR). The private key is stored securely. The CSR is delivered
to the Intermediate level which uses the CSR to create the End
Entity certificate. This certificate, along with the trust chain
back to the root, is then returned to the End Entity.

There is more to a pki, but this suffices for most development and
testing needs.


# Getting started and the Root level {#RootLevel}

This guide was developed on a Fedora 26 armv7hl system (Cubieboard2
SoC). It should work on most Linux and similar systems. All work
was done in a terminal window with extensive "cutting and pasting"
from a draft guide into the terminal window. Users of this guide
may find different behaviors based on their system.

## Setting up the Environment {#FirstStep}

The first step is to create the pki environment. Modify the
variables to suit your needs.


~~~~
export dir=/root/ca
export cadir=/root/ca
export format=pem
mkdir $dir
cd $dir
mkdir certs crl csr newcerts private
chmod 700 private
touch index.txt
touch serial
sn=8

countryName="/C=US"
stateOrProvinceName="/ST=MI"
localityName="/L=Oak Park"
organizationName="/O=HTT Consulting"
#organizationalUnitName="/OU="
organizationalUnitName=
commonName="/CN=Root CA"
DN=$countryName$stateOrProvinceName$localityName
DN=$DN$organizationName$organizationalUnitName$commonName
echo $DN
export subjectAltName=email:postmaster@htt-consult.com

~~~~

Where:

dir
: Directory for certificate files

cadir
: Directory for Root certificate files

Format
: File encoding:  PEM or DER <vspace />
  At this time only PEM works

sn
: Serial Number length in bytes <vspace />
  For a public CA the range is 8 to 19
{: hangIndent='9' vspace='0'}

The Serial Number length for a public pki ranges from 8 to 19
bytes. The use of 19 rather than 20 is to accommodate the hex
representation of the Serial Number. If it has a one in the high
order bit, DER encoding rules will place a 0x00 in front.

The DN and SAN fields are examples. Change them to appropriate
values. If you leave one blank, it will be left out of the
Certificate. "OU" above is an example of an empty DN object.

Create the file, $dir/openssl-root.cnf from the contents in {{Rootconfig}}.


## Create the Root Certificate {#RootCert}

Next are the openssl commands to create the Root certificate
keypair, and the Root certificate. Included are commands to view
the file contents.


~~~~
# Create passworded keypair file

openssl genpkey -aes256 -algorithm ec\
    -pkeyopt ec_paramgen_curve:prime256v1\
    -outform $format -pkeyopt ec_param_enc:named_curve\
    -out $dir/private/ca.key.$format
chmod 400 $dir/private/ca.key.$format
openssl pkey -inform $format -in private/ca.key.$format -text -noout

# Create Self-signed Root Certificate file
# 7300 days = 20 years; Intermediate CA is 10 years.

openssl req -config $dir/openssl-root.cnf\
     -set_serial 0x$(openssl rand -hex $sn)\
     -keyform $format -outform $format\
     -key $dir/private/ca.key.$format -subj "$DN"\
     -new -x509 -days 7300 -sha256 -extensions v3_ca\
     -out $dir/certs/ca.cert.$format

#

openssl x509 -inform $format -in $dir/certs/ca.cert.$format\
     -text -noout
openssl x509 -purpose -inform $format\
     -in $dir/certs/ca.cert.$format -inform $format

~~~~



# The Intermediate level {#IntermediateLevel}

## Setting up the Intermediate Certificate Environment {#NextStep}

The next part is to create the Intermediate pki environment.
Modify the variables to suit your needs.


~~~~
export dir=$cadir/intermediate
mkdir $dir
cd $dir
mkdir certs crl csr newcerts private
chmod 700 private
touch index.txt
sn=8 # hex 8 is minimum, 19 is maximum
echo 1000 > $dir/crlnumber

# cd $dir
commonName="/CN=Signing CA"
DN=$countryName$stateOrProvinceName$localityName$organizationName
DN=$DN$organizationalUnitName$commonName
echo $DN

~~~~

Create the file, $dir/openssl-intermediate.cnf from the contents in {{Intermediateconfig}}.


## Create the Intermediate Certificate {#IntermediateCert}

Here are the openssl commands to create the Intermediate
certificate keypair, Intermediate certificate signed request (CSR),
and the Intermediate certificate. Included are commands to view
the file contents.


~~~~
# Create passworded keypair file

openssl genpkey -aes256 -algorithm ec\
    -pkeyopt ec_paramgen_curve:prime256v1 \
    -outform $format -pkeyopt ec_param_enc:named_curve\
    -out $dir/private/intermediate.key.$format
chmod 400 $dir/private/intermediate.key.$format
openssl pkey -inform $format\
    -in $dir/private/intermediate.key.$format -text -noout

# Create the CSR

openssl req -config $cadir/openssl-root.cnf\
    -key $dir/private/intermediate.key.$format \
    -keyform $format -outform $format -subj "$DN" -new -sha256\
    -out $dir/csr/intermediate.csr.$format
openssl req -text -noout -verify -inform $format\
    -in $dir/csr/intermediate.csr.$format


# Create Intermediate Certificate file

openssl rand -hex $sn > $dir/serial # hex 8 is minimum, 19 is maximum
# Note 'openssl ca' does not support DER format
openssl ca -config $cadir/openssl-root.cnf -days 3650\
    -extensions v3_intermediate_ca -notext -md sha256 \
    -in $dir/csr/intermediate.csr.$format\
    -out $dir/certs/intermediate.cert.pem

chmod 444 $dir/certs/intermediate.cert.$format

openssl verify -CAfile $cadir/certs/ca.cert.$format\
     $dir/certs/intermediate.cert.$format

openssl x509 -noout -text -in $dir/certs/intermediate.cert.$format

# Create the certificate chain file

cat $dir/certs/intermediate.cert.$format\
   $cadir/certs/ca.cert.$format > $dir/certs/ca-chain.cert.$format
chmod 444 $dir/certs/ca-chain.cert.$format


~~~~


## Create a Server EE Certificate {#ServerCert}

Here are the openssl commands to create a Server End Entity
certificate keypair, Server certificate signed request (CSR),
and the Server certificate. Included are commands to view
the file contents.


~~~~
commonName=
DN=$countryName$stateOrProvinceName$localityName
DN=$DN$organizationName$organizationalUnitName$commonName
echo $DN
serverfqdn=www.example.com
emailaddr=postmaster@htt-consult.com
export subjectAltName="DNS:$serverfqdn, email:$emailaddr"
echo $subjectAltName
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:prime256v1\
    -pkeyopt ec_param_enc:named_curve\
    -out $dir/private/$serverfqdn.key.$format
chmod 400 $dir/private/$serverfqdn.$format
openssl pkey -in $dir/private/$serverfqdn.key.$format -text -noout
openssl req -config $dir/openssl-intermediate.cnf\
    -key $dir/private/$serverfqdn.key.$format \
    -subj "$DN" -new -sha256 -out $dir/csr/$serverfqdn.csr.$format

openssl req -text -noout -verify -in $dir/csr/$serverfqdn.csr.$format

openssl rand -hex $sn > $dir/serial # hex 8 is minimum, 19 is maximum
# Note 'openssl ca' does not support DER format
openssl ca -config $dir/openssl-intermediate.cnf -days 375\
    -extensions server_cert -notext -md sha256 \
    -in $dir/csr/$serverfqdn.csr.$format\
    -out $dir/certs/$serverfqdn.cert.$format
chmod 444 $dir/certs/$serverfqdn.cert.$format

openssl verify -CAfile $dir/certs/ca-chain.cert.$format\
     $dir/certs/$serverfqdn.cert.$format
openssl x509 -noout -text -in $dir/certs/$serverfqdn.cert.$format

~~~~


## Create a Client EE Certificate {#ClientCert}

Here are the openssl commands to create a Client End Entity
certificate keypair, Client certificate signed request (CSR),
and the Client certificate. Included are commands to view
the file contents.


~~~~
commonName=
UserID="/UID=rgm"
DN=$countryName$stateOrProvinceName$localityName
DN=$DN$organizationName$organizationalUnitName$commonName$UserID
echo $DN
clientemail=rgm@example.com
export subjectAltName="email:$clientemail"
echo $subjectAltName
openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:prime256v1\
    -pkeyopt ec_param_enc:named_curve\
    -out $dir/private/$clientemail.key.$format
chmod 400 $dir/private/$clientemail.$format
openssl pkey -in $dir/private/$clientemail.key.$format -text -noout
openssl req -config $dir/openssl-intermediate.cnf\
    -key $dir/private/$clientemail.key.$format \
    -subj "$DN" -new -sha256 -out $dir/csr/$clientemail.csr.$format

openssl req -text -noout -verify\
    -in $dir/csr/$clientemail.csr.$format

openssl rand -hex $sn > $dir/serial # hex 8 is minimum, 19 is maximum
# Note 'openssl ca' does not support DER format
openssl ca -config $dir/openssl-intermediate.cnf -days 375\
    -extensions usr_cert -notext -md sha256 \
    -in $dir/csr/$clientemail.csr.$format\
    -out $dir/certs/$clientemail.cert.$format
chmod 444 $dir/certs/$clientemail.cert.$format

openssl verify -CAfile $dir/certs/ca-chain.cert.$format\
     $dir/certs/$clientemail.cert.$format
openssl x509 -noout -text -in $dir/certs/$clientemail.cert.$format

~~~~



# The 802.1AR Intermediate level {#Intermediate8021ARLevel}

## Setting up the 802.1AR Intermediate Certificate Environment {#Step8021AR}

The next part is to create the 802.1AR Intermediate pki
environment. This is very similar to the Intermediate pki
environment. Modify the variables to suit your needs.


~~~~
export dir=$cadir/8021ARintermediate
mkdir $dir
cd $dir
mkdir certs crl csr newcerts private
chmod 700 private
touch index.txt
sn=8 # hex 8 is minimum, 19 is maximum
echo 1000 > $dir/crlnumber

# cd $dir
countryName="/C=US"
stateOrProvinceName="/ST=MI"
localityName="/L=Oak Park"
organizationName="/O=HTT Consulting"
organizationalUnitName="/OU=Devices"
#organizationalUnitName=
commonName="/CN=802.1AR CA"
DN=$countryName$stateOrProvinceName$localityName$organizationName
DN=$DN$organizationalUnitName$commonName
echo $DN
export subjectAltName=email:postmaster@htt-consult.com
echo $subjectAltName

~~~~

Create the file, $dir/openssl-8021AR.cnf from the contents in {{Intermediate8021ARconfig}}.


## Create the 802.1AR Intermediate Certificate {#Intermediate8021ARCert}

Here are the openssl commands to create the 802.1AR Intermediate
certificate keypair, 802.1AR Intermediate certificate signed
request (CSR), and the 802.1AR Intermediate certificate. Included
are commands to view the file contents.


~~~~
# Create passworded keypair file

openssl genpkey -aes256 -algorithm ec\
    -pkeyopt ec_paramgen_curve:prime256v1 \
    -outform $format -pkeyopt ec_param_enc:named_curve\
    -out $dir/private/8021ARintermediate.key.$format
chmod 400 $dir/private/8021ARintermediate.key.$format
openssl pkey -inform $format\
    -in $dir/private/8021ARintermediate.key.$format -text -noout

# Create the CSR

openssl req -config $cadir/openssl-root.cnf\
    -key $dir/private/8021ARintermediate.key.$format \
    -keyform $format -outform $format -subj "$DN" -new -sha256\
    -out $dir/csr/8021ARintermediate.csr.$format
openssl req -text -noout -verify -inform $format\
    -in $dir/csr/8021ARintermediate.csr.$format


# Create 802.1AR Intermediate Certificate file
# The following does NOT work for DER

openssl rand -hex $sn > $dir/serial # hex 8 is minimum, 19 is maximum
# Note 'openssl ca' does not support DER format
openssl ca -config $cadir/openssl-root.cnf -days 3650\
    -extensions v3_intermediate_ca -notext -md sha256\
    -in $dir/csr/8021ARintermediate.csr.$format\
    -out $dir/certs/8021ARintermediate.cert.pem

chmod 444 $dir/certs/8021ARintermediate.cert.$format

openssl verify -CAfile $cadir/certs/ca.cert.$format\
     $dir/certs/8021ARintermediate.cert.$format

openssl x509 -noout -text\
     -in $dir/certs/8021ARintermediate.cert.$format

# Create the certificate chain file

cat $dir/certs/8021ARintermediate.cert.$format\
   $cadir/certs/ca.cert.$format > $dir/certs/ca-chain.cert.$format
chmod 444 $dir/certs/ca-chain.cert.$format

~~~~


## Create an 802.1AR iDevID Certificate {#Cert8021AR}

Here are the openssl commands to create a 802.1AR iDevID
certificate keypair, iDevID certificate signed request (CSR), and
the iDevID certificate. Included are commands to view the file
contents.


~~~~
DevID=Wt1234
countryName=
stateOrProvinceName=
localityName=
organizationName="/O=HTT Consulting"
organizationalUnitName="/OU=Devices"
commonName=
serialNumber="/serialNumber=$DevID"
DN=$countryName$stateOrProvinceName$localityName
DN=$DN$organizationName$organizationalUnitName$commonName
DN=$DN$serialNumber
echo $DN

# hwType is OID for HTT Consulting, devices, sensor widgets
export hwType=1.3.6.1.4.1.6715.10.1
export hwSerialNum=01020304 # Some hex
echo  $hwType - $hwSerialNum

openssl genpkey -algorithm ec -pkeyopt ec_paramgen_curve:prime256v1\
    -pkeyopt ec_param_enc:named_curve\
    -out $dir/private/$DevID.key.$format
chmod 400 $dir/private/$DevID.$format
openssl pkey -in $dir/private/$DevID.key.$format -text -noout
openssl req -config $dir/openssl-8021AR.cnf\
    -key $dir/private/$DevID.key.$format \
    -subj "$DN" -new -sha256 -out $dir/csr/$DevID.csr.$format

openssl req -text -noout -verify\
    -in $dir/csr/$DevID.csr.$format
openssl asn1parse -i -in $dir/csr/$DevID.csr.pem
# offset of start of hardwareModuleName and use that in place of 189
openssl asn1parse -i -strparse 189 -in $dir/csr/$DevID.csr.pem

openssl rand -hex $sn > $dir/serial # hex 8 is minimum, 19 is maximum
# Note 'openssl ca' does not support DER format
openssl ca -config $dir/openssl-8021AR.cnf -days 375\
    -extensions 8021ar_idevid -notext -md sha256 \
    -in $dir/csr/$DevID.csr.$format\
    -out $dir/certs/$DevID.cert.$format
chmod 444 $dir/certs/$DevID.cert.$format

openssl verify -CAfile $dir/certs/ca-chain.cert.$format\
     $dir/certs/$DevID.cert.$format
openssl x509 -noout -text -in $dir/certs/$DevID.cert.$format
openssl asn1parse -i -in $dir/certs/$DevID.cert.pem

# offset of start of hardwareModuleName and use that in place of 493
openssl asn1parse -i -strparse 493 -in $dir/certs/$DevID.cert.pem

~~~~



# Footnotes {#Footnotes}

Creating this document was a real education in the state of
openSSL, X.509 certificate guidance, and just general level of
certificate awareness. Here are a few short notes.

## Certificate Serial Number {#SerNum}

The certificate serial number's role is to provide yet another way
to maintain uniqueness of certificates within a pki as well as a
way to index them in a data store. It has taken on other roles,
most notably as a defense.

The CABForum guideline for a public CA is for the serial number to
be a random number at least 8 octets long and no longer than 20
bytes. By default, openssl makes self-signed certificates with 8
octet serial numbers. This guide uses openssl's RAND function to
generate the random value and pipe it into the -set_serial option.
This number MAY have the first bit as a ONE; the DER encoding rules
prepend such numbers with 0x00. Thus the limit of '19' for the
variable 'ns'.

A private CA need not follow the CABForum rules and can use
anything number for the serial number. For example, the root CA
(which has no security risks mitigated by using a random value)
could use '1' as its serial number. Intermediate and End Entity
certificate serial numbers can also be of any value if a strong
hash, like SHA256 used here. A value of 4 for ns would provide a
sufficient population so that a CA of 10,000 EE certificates will
have only a 1.2% probability of a collision. For only 1,000
certificates the probability drops to 0.012%.

The following was proposed on the openssl-user list as an
alternative to using the RAND function:

Keep k bits (k/8 octets) long serial numbers for all your
certificates, chose a block cipher operating on blocks of k bits,
and operate this block cipher in CTR mode, with a proper secret key
and secret starting counter. That way, no collision detection is
necessary, you’ll be able to generate 2^(k/2) unique k bits longs
serial numbers (in fact, you can generate 2^k unique serial
numbers, but after 2^(k/2) you lose some security guarantees).

With 3DES, k=64, and with AES, k=128.


## subjectAltName support, or lack thereof {#SAN}

There is no direct openssl command line option to provide a
subjectAltName for a certificate. This is a serious limitation.
Per [RFC 2818](#RFC2818) SAN is the object for
providing email addresses and DNS addresses (FQDN), yet the common
practice has been to use the commonName object within the
distinguishedName object. How much of this is due to the
difficulty in creating certificates with a SAN?

Thus the only way to provide a SAN is through the config file. And
there are two approaches. This document uses an environment
variable to provide the SAN value into the config file. Another
approach is to use piping as in:

~~~~
openssl req -new -sha256 -key domain.key\
 -subj "/C=US/ST=CA/O=Acme, Inc./CN=foo.com" -reqexts SAN\
  -config <(cat /etc/ssl/openssl.cnf\
   <(printf "[SAN]\nsubjectAltName=DNS:foo.com,DNS:www.foo.com"))\
  -out domain.csr

~~~~



## DER support, or lack thereof {#DER}

The long, hard-fought battle with openssl to create a full DER pki
failed. The is no facility to create a DER certificate from a DER
CSR. It just is not there in the 'openssl ca' command. Even the
'openssl x509 -req' command cannot do this for a simple certificate.

Further, there is no 'hack' for making a certificate chain as there is
with PEM. With PEM a simple concatenation of the certificates create
a usable certificate chain. For DER, some recommend using
[PKCS#7](#RFC2315), where others point out that this format is poorly
support 'in the field', whereas [PKCS#12](#RFC7292) works for them.

Finally, openssl does supports converting a PEM certificate to DER:

~~~~
openssl x509 -outform der -in certificate.pem -out certificate.der

~~~~
This should also work for the keypair. However, in a highly
constrained device it may make more sense to just store the raw
keypair in the device's very limited secure storage.



# IANA Considerations {#IANA}

TBD. May be nothing for IANA.


# Security Considerations

Creating certificates takes a lot of random numbers. A good source
of random numbers is critical. Studies have found excessive amount
of certificates, all with the same keys due to bad randomness on
the generating systems. The amount of entropy available for these
random numbers can be tested. On Fedora/Centos use:

~~~~
cat /proc/sys/kernel/random/entropy_avail
~~~~


If the value is low (below 1000) check your system's randomness
source. Is rng-tools installed?  Consider adding an entropy
collection service like haveged from issihosts.com/haveged.

During the certificate creation, particularly during keypair
generation, the files are vulnerable to theft. This can be
mitigate using umask. Before using openssl, set umask:

~~~~
restore_mask=$(umask -p)
umask 077
~~~~
Afterwards, restore it with:

~~~~
$restore_mask
~~~~



# Acknowledgments

This work was jump started by the excellent RSA pki guide by Jamie
Nguyen. The openssl-user mailing list, with its many supportive
experts; in particular:  Rich Salz, Jakob Bolm, Viktor Dukhovni,
and Erwann Abalea, was of immense help as was the openssl man pages
website.

Finally, "Professor Google" was always ready to point to answers to
questions like: "openssl subjectAltName on the command line". And
the Professor, it seems, never tires of answering even trivial
questions.


--- back

# OpenSSL config files {#config}

## OpenSSL Root config file {#Rootconfig}

The following is the openssl-root.cnf file contents


~~~~
# OpenSSL root CA configuration file.
# Copy to `$dir/openssl.cnf`.

[ ca ]
# `man ca`
default_ca = CA_default

[ CA_default ]
# Directory and file locations.
dir               = $ENV::dir
cadir             = $ENV::cadir
format            = $ENV::format

certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand

# The root key and root certificate.
private_key       = $cadir/private/ca.key.$format
certificate       = $cadir/certs/ca.cert.$format

# For certificate revocation lists.
crlnumber         = $dir/crlnumber
crl               = $dir/crl/ca.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30

# SHA-1 is deprecated, so use SHA-2 instead.
default_md        = sha256

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 375
preserve          = no
policy            = policy_strict
copy_extensions   = copy

[ policy_strict ]
# The root CA should only sign intermediate certificates that match.
# See the POLICY FORMAT section of `man ca`.
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = optional

[ policy_loose ]
# Allow the intermediate CA to sign a more
#   diverse range of certificates.
# See the POLICY FORMAT section of the `ca` man page.
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = optional

[ req ]
# Options for the `req` tool (`man req`).
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only
req_extensions      = req_ext

# SHA-1 is deprecated, so use SHA-2 instead.
default_md          = sha256

# Extension to add when the -x509 option is used.
x509_extensions     = v3_ca

[ req_distinguished_name ]
# See <https://en.wikipedia.org/wiki/Certificate_signing_request>.
countryName                     = Country Name (2 letter code)
stateOrProvinceName             = State or Province Name
localityName                    = Locality Name
0.organizationName              = Organization Name
organizationalUnitName          = Organizational Unit Name
commonName                      = Common Name

# Optionally, specify some defaults.
# countryName_default             = US
# stateOrProvinceName_default     = MI
# localityName_default            = Oak Park
# 0.organizationName_default      = HTT Consulting
# organizationalUnitName_default  =

[ req_ext ]
subjectAltName = $ENV::subjectAltName

[ v3_ca ]
# Extensions for a typical CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
# keyUsage = critical, digitalSignature, cRLSign, keyCertSign
keyUsage = critical, cRLSign, keyCertSign
subjectAltName = $ENV::subjectAltName

[ v3_intermediate_ca ]
# Extensions for a typical intermediate CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
# keyUsage = critical, digitalSignature, cRLSign, keyCertSign
keyUsage = critical, cRLSign, keyCertSign

[ crl_ext ]
# Extension for CRLs (`man x509v3_config`).
authorityKeyIdentifier=keyid:always

[ ocsp ]
# Extension for OCSP signing certificates (`man ocsp`).
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning

~~~~


## OpenSSL Intermediate config file {#Intermediateconfig}

The following is the openssl-intermediate.cnf file contents


~~~~
# OpenSSL intermediate CA configuration file.
# Copy to `$dir/intermediate/openssl.cnf`.

[ ca ]
# `man ca`
default_ca = CA_default

[ CA_default ]
# Directory and file locations.
dir               = $ENV::dir
cadir             = $ENV::cadir
format            = $ENV::format

certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand

# The Intermediate key and Intermediate certificate.
private_key       = $dir/private/intermediate.key.$format
certificate       = $dir/certs/intermediate.cert.$format

# For certificate revocation lists.
crlnumber         = $dir/crlnumber
crl               = $dir/crl/ca.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30

# SHA-1 is deprecated, so use SHA-2 instead.
default_md        = sha256

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 375
preserve          = no
policy            = policy_loose
copy_extensions   = copy

[ policy_strict ]
# The root CA should only sign intermediate certificates that match.
# See the POLICY FORMAT section of `man ca`.
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = optional

[ policy_loose ]
# Allow the intermediate CA to sign a more
#  diverse range of certificates.
# See the POLICY FORMAT section of the `ca` man page.
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = optional
UID                     = optional

[ req ]
# Options for the `req` tool (`man req`).
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only
req_extensions      = req_ext

# SHA-1 is deprecated, so use SHA-2 instead.
default_md          = sha256

# Extension to add when the -x509 option is used.
x509_extensions     = v3_ca

[ req_distinguished_name ]
# See <https://en.wikipedia.org/wiki/Certificate_signing_request>.
countryName                     = Country Name (2 letter code)
stateOrProvinceName             = State or Province Name
localityName                    = Locality Name
0.organizationName              = Organization Name
organizationalUnitName          = Organizational Unit Name
commonName                      = Common Name
UID                             = User ID

# Optionally, specify some defaults.
# countryName_default             = US
# stateOrProvinceName_default     = MI
# localityName_default            = Oak Park
# 0.organizationName_default      = HTT Consulting
# organizationalUnitName_default  =

[ req_ext ]
subjectAltName = $ENV::subjectAltName

[ v3_ca ]
# Extensions for a typical CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
# keyUsage = critical, digitalSignature, cRLSign, keyCertSign
keyUsage = critical, cRLSign, keyCertSign

[ v3_intermediate_ca ]
# Extensions for a typical intermediate CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
# keyUsage = critical, digitalSignature, cRLSign, keyCertSign
keyUsage = critical, cRLSign, keyCertSign

[ usr_cert ]
# Extensions for client certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = client, email
nsComment = "OpenSSL Generated Client Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical,nonRepudiation,digitalSignature,keyEncipherment
extendedKeyUsage = clientAuth, emailProtection

[ server_cert ]
# Extensions for server certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "OpenSSL Generated Server Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[ crl_ext ]
# Extension for CRLs (`man x509v3_config`).
authorityKeyIdentifier=keyid:always

[ ocsp ]
# Extension for OCSP signing certificates (`man ocsp`).
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning

~~~~


## OpenSSL 802.1AR Intermediate config file {#Intermediate8021ARconfig}

The following is the openssl-8021ARintermediate.cnf file contents


~~~~
# OpenSSL 8021ARintermediate CA configuration file.
# Copy to `$dir/8021ARintermediate/openssl_8021AR.cnf`.

[ ca ]
# `man ca`
default_ca = CA_default

[ CA_default ]
# Directory and file locations.
# dir               = /root/ca/8021ARintermediate
dir               = $ENV::dir
cadir             = $ENV::cadir
format            = $ENV::format

certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand

# The root key and root certificate.
private_key       = $dir/private/8021ARintermediate.key.$format
certificate       = $dir/certs/8021ARintermediate.cert.$format

# For certificate revocation lists.
crlnumber         = $dir/crlnumber
crl               = $dir/crl/ca.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30

# SHA-1 is deprecated, so use SHA-2 instead.
default_md        = sha256

name_opt          = ca_default
cert_opt          = ca_default
default_enddate   = 99991231235959Z # per IEEE 802.1AR
preserve          = no
policy            = policy_loose
copy_extensions   = copy

[ policy_strict ]
# The root CA should only sign 8021ARintermediate
#   certificates that match.
# See the POLICY FORMAT section of `man ca`.
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = optional

[ policy_loose ]
# Allow the 8021ARintermediate CA to sign
#   a more diverse range of certificates.
# See the POLICY FORMAT section of the `ca` man page.
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = optional
serialNumber            = optional

[ req ]
# Options for the `req` tool (`man req`).
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only
req_extensions      = req_ext

# SHA-1 is deprecated, so use SHA-2 instead.
default_md          = sha256

# Extension to add when the -x509 option is used.
x509_extensions     = v3_ca

[ req_distinguished_name ]
# See <https://en.wikipedia.org/wiki/Certificate_signing_request>.
countryName                     = Country Name (2 letter code)
stateOrProvinceName             = State or Province Name
localityName                    = Locality Name
0.organizationName              = Organization Name
organizationalUnitName          = Organizational Unit Name
commonName                      = Common Name
serialNumber                    = Device Serial Number

# Optionally, specify some defaults.
0.organizationName_default      = HTT Consulting
organizationalUnitName_default  = Devices

[ req_ext ]
subjectAltName = otherName:1.3.6.1.5.5.7.8.4;SEQ:hmodname

[ hmodname ]
hwType = OID:$ENV::hwType
hwSerialNum = FORMAT:HEX,OCT:$ENV::hwSerialNum

[ v3_ca ]
# Extensions for a typical CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ v3_8021ARintermediate_ca ]
# Extensions for a typical
#   8021ARintermediate CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
# keyUsage = critical, digitalSignature, cRLSign, keyCertSign
keyUsage = critical, cRLSign, keyCertSign

[ 8021ar_idevid ]
# Extensions for IEEE 802.1AR iDevID
#   certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment

[ crl_ext ]
# Extension for CRLs (`man x509v3_config`).
authorityKeyIdentifier=keyid:always

[ ocsp ]
# Extension for OCSP signing certificates (`man ocsp`).
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning

~~~~
