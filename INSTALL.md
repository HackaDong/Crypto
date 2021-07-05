# Section 1: System Configuration
**ITEM** | **ATTRIBUTION**
---|:--:
Process |	Intel(R) Xeon(R) Platinum 8280L CPU @ 2.70GHz  
Socket |	2  
Cores Per Processor |	28  
Logical Process |	56  
Platform |	Purley  
Accelerator |	Lewisburg  
OS |	Ubuntu 18.04.4 LTS  
OS/Kernel Comments |	4.15.0-102-generic  
GCC version | 7.5.0  
CMAKE version | 3.10.2  
QAT | QAT1.7.L.4.14.0-00031  
OPNESSL | openssl-1.1.1k  
QAT_Engine | v0.6.6  
Async_mode_nginx | v0.4.5  
Nginx-QUIC | changeset: 7518:ac0398da8  
BoringSSL | commit 5e7229488844e987207b377968b3cf0340bc4ccf  
	
# Section 2: BIOS Setup
Refer to: https://cat.intel.com/Benchmarks/NGINX_OpenSSL_WebServer_SKX_8160_24C-2100MHz_Ubuntu18_04.pdf  
**Note**: TURBO mode must be disabled, and set BOOT PERFORMANCE MODE as MAX_PERFORMANCE
```
Tips: Check the CPU MHZ to make sure the TURBE is closed
$ watch -n.1 "grep \"^[c]pu MHz\" /proc/cpuinfo"
```

# Section 3: Server Software Stack Installation
## QAT Driver
**Reference**: https://01.org/intel-quickassist-technology  
### REQUIREMENT  
* check hardware existence: `$ lspci -vnd :37c8`, commonly there will be **3** qat devices  
* necessary pre-install:   
```
$ apt-get update
$ apt-get install pciutils-dev g++ pkg-config libssl-dev yasm libboost-all-dev libnl-3-dev libnl-genl-3-dev
```
### INSTALL  
```
$ mkdir qat_driver
$ cd qat_driver
$ wget https://downloadcenter.intel.com/downloads/eula/30178/Intel-QuickAssist-Technology-Driver-for-Linux-HW-Version-1-7?httpDown=https%3A%2F%2Fdownloadmirror.intel.com%2F30178%2Feng%2FQAT1.7.L.4.14.0-00031.tar.gz
$ tar xf <xxx.tar.gz>
$ ./configure
$ make -j && make install -j
```
### UTILS  
```
# check the kernel module
$ lsmod|grep qat
# check the qat status
$ adf_ctl status
# restart qat driver
$ adf_ctl restart
```
### Config File  
Each qat device has its own config file, which located in **/etc/c6xx_devX.conf**  
```
ServicesEnabled: cy;dc -> cy
ServicesProfile: DEFAULT -> CRYPTO (in case we need to offload HKDF)
[SSL] -> [SHIM]
NumberCyInstances: 1
NumberDcInstances: 0
NumProcesses: 32
LimitDevAccess: 0
```
**NOTE**: `adf_ctl restart` should be executed every time you change the config file  
**Tips**: change the config files in <QAT_DIR>/quickassist/utils/adf_ctl/conf_files, then `make install`, which is a permanent and easy way  

## BoringSSL
**Reference**: https://boringssl.googlesource.com/boringssl/  
**BUILD**: https://boringssl.googlesource.com/boringssl/+/HEAD/BUILDING.md
### Pre-Requirement
* CMake 3.5 or later is required.
* GCC version 4.8.5 met some error while building, upgrade to 8.3.0 works well
* GO is installed
* libunwind is optional
### INSTALL
```
# upstream version
$ git clone git clone https://boringssl.googlesource.com/boringssl
# X22519 Optimized version
$ git clone ssh://git@gitlab.devtools.intel.com:29418/oppo-quic-nginx/boringssl.git

$ cd boringssl
$ mkdir build
$ cd build
$ cmake ..
$ make
```

## QUIC-ENGINE
This is a library which can offload asymmetric crypto operations to the QAT within BoringSSL archetecture.  
Three main exported functions:  
* quic_ssl_priv_sign
* quic_ssl_priv_dec
* quic_ssl_complete
### INSTALL
```
$ git clone ssh://git@gitlab.devtools.intel.com:29418/oppo-quic-nginx/quic_engine.git
$ cd quic_engine
# set related environment
$ export BORINGSSL_DIR=<>
$ export BORINGSSL_BUILD_DIR=<>
$ export QAT_DIR=<>
$ make
```
It will generation **libquicengine.a** static library for use.  
### TEST
Need to modify the Makefile to build the executable file to run the test.  
Remember to recovery it to static library version after testing.  
```
# in Makefile
ALL: libquicengine.a -> main
$(ALL): $(OBJS): 
          $(AR) $(ARFLAGS) $(ALL) $(OBJS) 
          -> 
             $(CC) $(LDFLAGS) $(OBJS) $(LIBS) -o $(ALL)

$ make
$ ./main
``` 

## NGINX-QUIC
**Reference**: https://hg.nginx.org/nginx-quic/shortlog  
**BUILD**: README.md
### INSTALL
```
# upstream version
$ hg clone -b quic https://hg.nginx.org/nginx-quic
# Using QAT version
$ git clone ssh://git@gitlab.devtools.intel.com:29418/oppo-quic-nginx/quic_nginx.git

$ cd <nginx_quic_dir>
# upstream version
$ ./auto/configure \
        --prefix=<nginx_install_dir> \
        --with-http_v3_module \
        --with-cc-opt="-I<boringssl_dir>/include" \
        --with-ld-opt="-L<boringssl_dir>/build/ssl \
                       -L<boringssl_dir>/build/crypto"

# Using QAT version:
$ export QAT_DIR=<qat_driver_dir>
$ make clean
$ ./auto/configure \
        --prefix=<nginx_install_dir> \
        --with-http_v3_module \
        --with-cc-opt="-I<boringssl_dir>/include" \
                       -I<quic_engine_dir> \
                       -I$QAT_DIR/quickassist/include \
                       -I$QAT_DIR/quickassist/include/lac \
                       -I$QAT_DIR/quickassist/include/dc \
                       -I$QAT_DIR/quickassist/lookaside/access_layer/include \
                       -I$QAT_DIR/quickassist/utilities/libusdm_drv \
                       -I$QAT_DIR/quickassist/lookaside/access_layer/src/sample_code/functional/include" \
        --with-ld-opt="-L<boringssl_dir>/build/ssl \
                       -L<boringssl_dir>/build/crypto \
                       -L$QAT_DIR/build \
                       -L<quic_engine_dir> \
                       -lqat_s -lusdm_drv_s -lquicengine"
# make -j
# make install -j
```
### NGINX config file
Located in <nginx_install_dir>/conf/nginx.conf  
Refer to: ssh://git@gitlab.devtools.intel.com:29418/oppo-quic-nginx/nginx_config_files.git

### certification and private key
```
# generate rsa 2k private key and self-signed certification
$ openssl req -newkey rsa:2048 -nodes -keyout rsa_private.key -x509 -days 365 -out cert.crt
```

# Section 4: Client
## ab
$ ./ab -n 100000 -c 100  https://10.67.115.25:443/  
## quic client
Refer to oppo's doc.
