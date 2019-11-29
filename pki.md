# Setting up the Public Key Infrastructure

The PKI is based on Cloudflare's excellent cfssl tool.

Older versions of this tool are available for download from cloudflare's
Web site, but to get to the latest version, you have to compile from scratch.

Fortunately, this is very easy to do. You will need a working Go language
environment on your workstation (not on one of the cluster systems). On RedHat-like systems, simply install the following RPMs: golang golang-bin.

Run the following script in an empty directory. It will download the cfssl
source code, compile it for you and copy the three files you need into the root
of the same empty directory.

```
export GOPATH=${PWD}
git clone https://github.com/cloudflare/cfssl.git $GOPATH/src/github.com/cloudflare/cfssl
pushd $GOPATH/src/github.com/cloudflare/cfssl
make
popd

cp $GOPATH/src/github.com/cloudflare/cfssl/bin/{cfssl,cfssljson,multirootca} ${PWD}
```

