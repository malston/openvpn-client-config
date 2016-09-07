Use this to generate client certs for [openvpn-bosh-release](https://github.com/dpb587/openvpn-bosh-release/releases).

Each client that needs to connect should have their own client key. Clients should generate their own keys and send a certificate signing request to the operator for them to verify and then return a signed certificate.

## Setup

Customize an environment file to store some reusable config values, then load it...

    $ cat > .env << EOF
    export EASYRSA_VARS_FILE="vars"
    export EASYRSA_REQ_COUNTRY="US"
    export EASYRSA_REQ_PROVINCE="Colorado"
    export EASYRSA_REQ_CITY="Denver"
    export EASYRSA_REQ_ORG="Copyleft Certificate Co"
    export EASYRSA_REQ_OU="My Organizational Unit"
    export EASYRSA_REQ_EMAIL="me@example.net"
    EOF
    $ source .env

## Key Generation

To create a key, the client should generate a key and CSR...

    # a temporary directory to work in
    $ mkdir tmp-myovpn
    $ cd tmp-myovpn

    # a unique name to represent this client key
    $ TMP_CN=$(hostname -s)-$(date +%Y%m%da)

    # create the key and csr
    $ openssl req -new -nodes -days 3650 -newkey rsa:2048 \
      -subj "/C=$EASYRSA_REQ_COUNTRY/ST=$EASYRSA_REQ_PROVINCE/L=$EASYRSA_REQ_CITY/O=$EASYRSA_REQ_ORG/OU=$EASYRSA_REQ_OU/CN=$TMP_CN/emailAddress=`git config user.email`" \
      -out "$TMP_CN.req" \
      -keyout openvpn.key

The operator can then [sign the request](#signing-client-requests).


## Client Configuration

Generally, OpenVPN clients will need to use a `*.ovpn` profile to establish a connection to the OpenVPN server. The profile contains connection parameters and authentication information.

The operator can provide the following configuration to the client...

    client
    dev tun
    proto tcp
    remote {hostname-or-ip} {port}
    comp-lzo
    resolv-retry infinite
    nobind
    persist-key
    persist-tun
    mute-replay-warnings
    remote-cert-tls server
    verb 3
    mute 20
    tls-client
    <ca>
    {ca-certificate-data}
    </ca>
    <cert>
    {client-certificate-signed-by-operator-data}
    </cert>

And the client should then append their own key...

    <key>
    {client-private-key-data}
    </key>

Once appended, the client can use the profile or import it to a GUI...

    $ openvpn --config client.ovpn

## Signing Client Requests

When the client sends a CSR to the operator, the operator can sign the request using the following...

    $ cd pki
    $ cp $TMP_CN.req ./reqs/$TMP_CN.req
    $ ../easyrsa/easyrsa sign client "$TMP_CN"

## VPN Client Configuration

Once you have a signed CSR from the operator, you can use the file data for your VPN [client configuration](client.ovpn.erb).

    * {ca-certificate-data} -> ./pki/ca.crt
    * {client-certificate-signed-by-operator-data} -> ./pki/issued/$TMP_CN.crt
    * {client-private-key-data} -> ./tmp-myovpn/openvpn.key
