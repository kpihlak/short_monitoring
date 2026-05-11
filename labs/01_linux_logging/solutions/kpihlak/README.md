\# Linux logging lab



\## LogServer



\### rsyslog install



```bash

sudo apt update

sudo apt install -y rsyslog

```



\### UDP/TCP vastuvõtt



Lisa faili `/etc/rsyslog.conf`



```conf

module(load="imudp")

input(type="imudp" port="514")



module(load="imtcp")

input(type="imtcp" port="514")

```



\### Kauglogide kaust



```bash

sudo mkdir -p /var/log/remote

sudo chown syslog:adm /var/log/remote

```



Fail `/etc/rsyslog.d/remote.conf`



```conf

\*.\* /var/log/remote/syslog.log

```



Restart:



```bash

sudo systemctl restart rsyslog

```



Kontroll:



```bash

sudo ss -tuln | grep 514

```



\---



\## LogClient



Fail `/etc/rsyslog.d/forward.conf`



```conf

\*.\* @@192.168.1.186:514

```



Restart:



```bash

sudo systemctl restart rsyslog

```



Kontroll:



```bash

logger TESTLOG

```



Serveris:



```bash

tail -f /var/log/remote/syslog.log

```

