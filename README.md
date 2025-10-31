# annelids-multiplayer
this repository was created to make a conclusion on what i learned and found about Annelids servers.
in the future i might make this repository open if i will get bored or game will die, so if you want to fully understand this page make sure to have knowledge of:
- sockets, ports, webpages
- python 3 and socket library
- linux terminal/termux
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
after launching the program i gor this response:
```
'\x16\x02\x00\x00\x03\x00\n&\x18\xf7\x80\x04M<@\x00\x00"\x01-*\x12maps/corridors.map0\x028\x00@\x06\n!\x18\xf7\x80\x04M<$\x00\x00"\x01-*\rmaps/xmas.map0\x068\x00@\x06\n"\x18\xf7\x80\x04M<L\x00\x00"\x04100k*\x0bmaps/2D.map0\x018\x04@\x06\n#\x18\xf7\x80\x04M<,\x00\x00"\x01-*\x0frandom_oblivion0\t8\x00@\x06\n#\x18\xf7\x80\x04M<T\x00\x00"\x01-*\x0frandom_oblivion0\t8\x04@\x06\n\x1f\x18\xf7\x80\x04M<4\x00\x00"\x01-*\x0brandom_city0\x018\x04@\x06\n \x18\xf7\x80\x04M<\x0c\x00\x00"\x01-*\x0crandom_pipes0\x018\x05@\x06\n\x1e\x18\xf7\x80\x04M<\x18\x00\x00"\x01-*\nrandom_ice0\x018\x06@\x06\n\x1e\x18\xf7\x80\x04M<\x14\x00\x00"\x01-*\nrandom_ice0\x018\x01@\x06\n"\x18\xf7\x80\x04M<\x08\x00\x00"\x01-*\x0erandom_electro0\x048\x00@\x06\n\x1e\x18\xf7\x80\x04M<\x04\x00\x00"\x01-*\nrandom_sea0\x088\x00@\x06\n#\x18\xf7\x80\x04M<\x00\x00\x00"\x01-*\x0fmaps/igloos.map0\x038\x00@\x06\n\x1f\x18\xf7\x80\x04M<l\x00\x00"\x01-*\x0brandom_city0\x058\x02@\x06\n&\x18\xf7\x80\x04M<D\x00\x00"\x01-*\x12maps/corridors.map0\x058\x00@\x06\n \x18\xf7\x80\x04M< \x00\x00"\x01-*\x0crandom_space0\x078\x00@\x06'
```
yea its not really readable right?

however after deeper look we can see that it actually sends info for every room in one byte. look closer:
```
\x12maps/corridors.map0\x058\x00@\x06\n \x18\xf7\x80\x04M<
```
**maps/corridors.map** obviously stands for the map. there is enumerator in game code which accepts raw map name and returns in-game map name like this:
```
random_pipes
>>> Labyrinth
```
so i decided to just make dictionary like this and then use it in my code:
```
k = map_enum.keys()
	for j in k:
		if j in i:
			s = f"{s}    {map_enum[j]}"
```
we're getting all keys of this dictionary, checking if there is any key from this dictionary in the line, and then adding it into our string
**\x058** - its actually gamemode in the room. if you will look at every line you can see that this byte differs. so we can try and parse that!
```
d=d.split('"')
for i in d:
	if "x018" in i:
		if "100k" in i:
			s = "deathmatch (100 kills)"
		else:
			s = "deathmatch"
	elif "x028" in i:
		s = "team deathmatch"
	elif "x038" in i:
		s = "ctf"
	elif "x048" in i:
		s = "conq"
	elif "x058" in i:
		s = "koth"
	elif "x068" in i:
		s = "egg"
	elif "x078" in i:
		s = "team egg"
	elif "x088" in i:
		s = "crown"
	elif "t8" in i:
		s = "zombie"
```
in this code we splitted lines by quote into the list, then we parsed the lines and created a string to store gamemode of every room! now our program will automatically find gamemode for every room!
but we still dont know amount of players in the room. if you will look once more on the line you can spot **"\x00@\x06"** bytes. as you already guessed, its exactly what we need!
```
    ind = i.find("@")
	cnt = i[ind-2:ind+5].replace("x", "").replace("@", "")
	print(s + "    " + cnt)
```
here we find index of these bytes, then removing "x" and "@" from them, and printing the result!
now we can take our line that we firstly mentioned and turn it in the actual room description!
```
\x12maps/corridors.map0\x058\x00@\x06\n \x18\xf7\x80\x04M<
>>> King of the hill
>>> Corridors
>>> 00/06
```
now we can easily get the list of available games!

you might also notice than there is two servers in the output. After some digging i found out that there is actually servers for Spain (ms.annelids.io) and Russia (api.annelids.io). 
## Getting inside of room
Alright, we know how to view available games. But can we view only one game that we needed?

## Epilogue
If you see this, thank you very much for reading! I actually worked really hard to get all this info. If you liked this, you might want to join my telegram channel @annelidss!
