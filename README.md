This guide provides instructions and support files to run the AFLnet fuzzer on the [Kamailio SIP](https://www.kamailio.org/) network server, on Ubuntu 18.04.2 LTS.

The guide assumes that AFLnet has already been compiled according to its [tutorial](https://github.com/aflnet/aflnet#tutorial---fuzzing-live555-media-streaming-server), and that environmental variables (e.g., `$AFLNET`) are set accordingly.

Moreover, the guide assumes that the `$WORKDIR` variable has been set to the folder with a copy of this repository, as follows:

```
git clone https://github.com/rnatella/aflnet-kamailio-sip.git
cd aflnet-kamailio-sip
export WORKDIR=$(pwd)
```


AFLnet must be re-compiled with support to parse SIP requests and responses:

```
cd $AFLNET
patch -p1 < $WORKDIR/aflnet-kamailio-sip.patch
make
```


To fuzz the SIP server, we will be simulating two SIP users that make a call (`REGISTER + INVITE`, adapted from this [tutorial](https://tomeko.net/other/sipp/sipp_cheatsheet.php)). One of the users will be the fuzzer; the other user will be a SIP client ([PJSIP](https://www.pjsip.org/)).

![SIP scenario](/sipp_pjsua.png)


Before fuzzing, we test the Kamailio SIP server under a normal scenario. We patch the server to prevent unnecessary child processes, and to disable its random number generator. To build the server (commands are from the [installation instructions from the GIT repo of the project](https://github.com/kamailio/kamailio/blob/master/INSTALL)):

```
cd $WORKDIR

git clone https://github.com/kamailio/kamailio.git

cd kamailio

patch -p1 < $WORKDIR/kamailio.patch

make cfg
make all
make install
```


We patch the PJSIP client to disable its random number generator, and build it:

```
cd $WORKDIR

git clone https://github.com/pjsip/pjproject.git

cd pjproject/

patch -p1 < $WORKDIR/pjsip.patch

./configure
make dep && make clean && make

```


To test the SIP server before fuzzing, we use an additional SIP client ([SIPp](https://github.com/SIPp/sipp)). To build the SIPp client:

```
cd $WORKDIR

git clone https://github.com/SIPp/sipp.git

cd sipp

sudo apt install -y pkg-config dh-autoreconf ncurses-dev build-essential libssl-dev libpcap-dev libncurses5-dev libsctp-dev lksctp-too
ls

./build.sh --full
```

To configure the server, we use a customized version of the `kamailio-basic.cfg` sample configuration file, where we disable forking, only enable one listening port (on UDP `5060`), and remove unnecessary modules that require external databases and configurations (ACCOUNTING, AUTH, PERMISSION, TLS) and for advanced networking (NAT).

To run the server:

```
cd $WORKDIR/kamailio

mkdir runtime_dir/

./src/kamailio -f $WORKDIR/kamailio-basic.cfg -L ./src/modules -Y ./runtime_dir/ -n 1 -D  -E
```

Then, to run the PJSIP client:

```
cd $WORKDIR/pjproject

./pjsip-apps/bin/pjsua-x86_64-unknown-linux-gnu --local-port=5068 \
                --id sip:33@127.0.0.1 --registrar sip:127.0.0.1 \
                --proxy sip:127.0.0.1 --realm '*' --username 33 --password 33 \
                --auto-answer 200 --auto-play --play-file $WORKDIR/StarWars3.wav --auto-play-hangup \
                --duration=10
```

Finally, to run the SIPp client:

```
cd $WORKDIR/sipp

sipp 127.0.0.1 -sf $WORKDIR/sipp_scenario/REGISTER_INVITE_client2.xml -inf $WORKDIR/sipp_scenario/REGISTER_INVITE_client.csv -m 1 -l 1 -r 1 -rp 1000
```


The output of SIp should appear as follows:

```
                                 Messages  Retrans   Timeout   Unexpected-Msg
    REGISTER ---------->         1         0         0
         200 <----------         1         0         0         0
      INVITE ---------->         1         0         0
         100 <----------         1         0         0         0
         180 <----------         0         0         0         0
         183 <----------         0         0         0         0
         200 <----------         1         0         0         0
         ACK ---------->         1         0
       Pause [    500ms]         1                             0
         BYE ---------->         1         0         0
         200 <----------         1         0         0         0
```



To fuzz the server, we will launch both the SIP server, and the PJSIP client, using the `-c` option of AFLNet to launch our `pjsip.sh` script before communicating with the server. 

```
cd $WORKDIR

$AFLNET/afl-fuzz -d -i in-sip/ -o out-sip/ -m 200 -N udp://127.0.0.1/5060 -l 5061 -P SIP -D 50000 -t 3000+  -q 3 -s 3 -E -K -R  -c ./pjsip.sh  -- ./kamailio/src/kamailio -f ./kamailio-basic.cfg -L ./kamailio/src/modules -Y ./kamailio/runtime_dir/ -n 1 -D -E 
```

Please note that to run the seed provided with this tutorial, you need to have AFLNet connect from the `5061` UDP source port, using the input parameter `-l` added by the patch.

