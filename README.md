# annelids-multiplayer
this repository was created to make a conclusion on what i learned and found about Annelids servers.
in the future i might make this repository open if i will get bored or game will die
## Capturing packets
the first thing i started doing was capturing and analysing game packets. it was easily done via Packet Capture app cause game doesnt use SSL pinging.

as we can see in the screenshot annelids sends packets to 255.255.255.255, 209.222.4.172, 51.15.202.216 IP addresses.
the first IP doesnt give any response and is not likely related to Annelids servers. however, other IPs actually registered like domains:
```
51.15.202.216 - 216-202-12-51.instances.scw.cloud - api.annelids.io
209.222.4.172 - 209.222.4.172.static.afterburst.com - ms.annelids.io
```
which means that annelids uses virtual servers located somewhere in france
if we look again at screenshot we can see that it send same packets on both IPs on port 65532, which is probably used to just check if servers is up
next it sends packet on api.annelids.io on port 12360. this is where interesting part starts:

as we can see game sends two requests on the server and then continuously gets list of available games. however the only thing that we can read in here is map names, rest is not understandable.
i decided to write small program on python to repeat game's requests:
```
import socket

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
	sock.connect(("api.annelids.io", 12360))
	sock.send(bytes.fromhex("12 00 00 00 01 00"))
	sock.send(bytes.fromhex("08 3c 10 f7 80 04 18 ea e1 04 28 00"))
	data = sock.recv(65536)
	d = repr(data)
	print(d)
```
ill assume you already know what python and socket library is, so i wont describe that
however you can think: "what do we even sending to this server?"
all of this i took from the Packet Capture app, because its actually stores requests in hex which is really useful for replicating game behaviour 

while making this page i found out that some bytes actually differs in second packet ("08 3c 10 f7 80 04 18 **ea e1 04** 28 00")
even although it doesnt affect next packets that might relate to something, you can try and find it by yourself
## understanding response
