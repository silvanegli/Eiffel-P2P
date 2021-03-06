Readme file for p2p-client
==========================

- team: "Silvan Egli, Simon Peyer"
- previous contributors: "Qing Cheng"
- supervisor: "Jocelyn Fiat"
- date: "2015-jan-14"

### 1. Introduction
---------------
NAT stands for Network Address Translation was introduced to solve the problem of the
internet running out of addresses. It can for example be implemented on a router in
a home network. The router has a public ip-address, lets assume 190.52.187.2. Now it 
can assign new (private )ip-addresses to devices in its network.  Assume a laptop gets assigned
10.0.0.1. When the laptop sends a packet from port 4111 out of the home network the 
router replaces the source address and source port (10.0.0.1/4111) with it's public ip
and a new port e.g (190.52.187.2/60344) it remembers this mapping (10.0.0.1/4111 -> 190.52.187.2/60344)
in a table. When a packet for the laptop comes in from the internet the router replaces
the destination address and port according to the mapping. i.e replaces  190.52.187.2/60344
with 10.0.0.1/4111

A disadvantage of NAT is that the laptop (generally all devices in a network with NAT) 
can't act as a public server. The reason is that the laptop has no public ip-address anymore as the router
can assign arbitrary private ip-addresses to it.

The avoiding technique used for NAT in this project is UDP Hole Punching and will be explained
later in this document.

This project is an interface written in Eiffel that enables two clients to connect through a Rendezvous-Server.
With this tool it is possible to establish a UDP connection between two clients.
As a result the clients are able to send strings to each other.


### 2. Idea
----------------

We are providing two tools:
- a) Rendezvous-Server
- b) Client-Interface

a)
The Rendezvous server is in charge to store the public IP, port and username of a specific user.
It always provides, a function to get a list of all users which are in the database of the server, as well 
as a lookup function to get the IP address and port for a specific user.
The Rendezvous Server must be deployed in the public internet meaning it must have a well known 
public IP and a port and must be accessible on that.

b)
The Client is interested to establish a connection to another Client.
In this example Client_1 tries to connect to Client_2.
We assume such a connection:

(private: 193.0.0.2/40001) 		(public: 188.4.51.191)		(public	201.2.68.74/8888)		(public: 194.18.15.51)		(private: 10.0.0.1/40001)
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Client_1 --------------------	NAT_1  -------------------	Rendezvous	------------------	NAT_2  --------------------- Client_2
															          Server

In a first stage, the Client_1 and Client_2 will register themselves to the Rendezvous server.
After registering the entries of the NAT-tables and the database of the server look as following

	NAT_1 Translation Table:						
	193.0.0.2/40001	<-> 188.4.51.191/ 50057	
	
	NAT_2 Translation Table
	10.0.0.1/40001 <-> 194.18.15.51/61442
	
	Database:
	Client_1:	188.4.51.191/ 50057	
	Client_2:	194.18.15.51/61442

The Server has now public IP,PORT,User_name of Client_1 and Client_2 in its database.
Therefore Client_1 can now ask for the userlist of the Rendezvous-Server, this will give him back,
among others, the username of the Client_2.
With this username he can now ask the server for the public IP-address and port of Client_2.
So Client_1 has now IP,Port, username of client_2.
And vice versa, Client_2 can do the same for Client_1.									

In a second stage, the Client_1 and Client_2 connect to each other using UDP_HolePunch.

This works as following:
Now that both endpoints have public IP/port of the other they start sending UDP Packets
to each other. They must both send because another issue of NAT devices is that they
often won't let a packet pass from the internet to the private network without having 
seen an outgoing packet with the same IP/port before. When a NAT device sees a packet
going out to the internet it creates a rule. This means it remembers the mapping
(in the Translation Table) and also the source and destination IP/port. By sending out
a UDP packet we "punch a hole" in the NAT device such that packets from the other
peer can come in. Therefore the name UDP Hole Punch

Finally if both peers have received a UDP packet from the other one they are connected.
   
	Client_1 <--------> Client_2

In UDP there is no explicit connection teardown like in TCP. So the NAT's generally
don't know when a rule won't be used anymore. Therefore they have Idle Time-outs after
which the rule is deleted. To avoid this both endpoints have to send so called keep-alive
packets to each other periodically.


### 3. Requirements
---------------
First there are needed two clients and one server which is running on public IP/port.
Both the Rendezvous_Server and the Client_Interface make use of the Eiffel net library.
Furthermore the Client_Interface uses the Eiffel thread and time library.
For running multiple threads the Concurrency mode of the Client_Interface project
must be set to EiffelThread in the project settings (or in the .ecf file).
Additionally the project must use a multithreaded precompiled library. This
can also be changed in the project settings by setting the Location of the precompiled
library to $ISE_PRECOMP\base-mt-safe.ecf or by setting the following entry in the .ecf file:
<precompile name="base_pre" location="$ISE_PRECOMP\base-mt-safe.ecf"/>

As format of the UDP packets JSON was chosen. Therefore the ejson library from
https://github.com/eiffelhub/json was integrated and used in a separate cluster.
The format of the different packets can be found in a separate file called Json_Client.txt

The CONNECTION_MANAGER.connect feature must be called on both sides within the
given connecting_timeout.


4. Contents
-----------
Classes of Rendezvous_Server:

	APPLICATION:		This is the root class. The server listens in an endless loop for incoming packets. When a packet
				arrives it parses it to a JSON_OBJECT, detects the message type and passes the JSON_OBJECT to 
				the corresponding handler.
	
	CLIENT_DATABASE:	This class provides an interface for the APPLICATION class. It is client of HASH_TABLE[NETWORK_SOCKET_ADDRESS, STRING]
				which is the implementation of the database. The different features are the interface to this
				HASH_TABLE where the public IP/Port for each user are stored.
	
	UTILS:			This static class provides the constant that are necessary for the p2p protocol to run. Therefore a lot must be 
				equal to the UTILS class of the Client_Interface. For example the error_type constants.
						
Classes of Client_Interface:

	TEST:			When set as root class TEST was and can be used to test the Client_Interface. It is not necessary for
				the Client_Interface to run.
	
	CONNECTION_MANAGER:	This class is the interface for a project using Client_Interface. The following features can be used:
	
					register(a_name: STRING): BOOLEAN		
							Sends a packet to the server indicating to register a user with username a_name.
							It returns whether the registering succeeded or not. What can go wrong:
							server_down : the server did not respond within the timeout
							client_already_registered: there exists already an identical entry in the server database
							client_name_already_used: there exists an entry with the same name but other public IP in the server database
							unknown_error: an error that could not be handled occurred
					unregister(a_name: STRING): BOOLEAN
							Like register. What can go wrong:
							server_down : the server did not respond within the timeout
							client_not_registered: there is no user with a_name in the database
							invalid_unregister_attempt: the IP for a_name in the database did not match with the IP that tried to unregister a_name
							unknown_error: an error that could not be handled occurred	
					get_registered_users: BOOLEAN
							Asks the server to hand out the currently registered users. Returns whether it succeeded.
							If succeeded then the currently registered users can be found in registered_users as ARRAY of STRING. What can go wrong:
							server_down : the server did not respond within the timeout
							unknown_error: an error that could not be handled occurred	
					connect(a_peer_name: STRING): BOOLEAN
							Tries to connect to a_peer_name and returns whether the connection was established or not.
							First it queries the server for the public IP/Port of a_peer_name and if that succeeded it goes on with the 
							hole punching. What can go wrong:
							server_down : the server did not respond the IP/Port query within the timeout
							client_not_registered: there is no user with a_peer_name in the database
							client_not_responding: no udp_hole_punch message of the other peer was received
							unknown_error: an error that could not be handled occurred	
					start
							This feature launches the UDP_RECEIVE_THREAD and UDP_SEND_THREAD and should therefore be called before any other feature.
					stop
							Forces UDP_RECEIVE_THREAD, UDP_SEND_THREAD and KEEP_ALIVE_THREAD to finish by setting a finish flag. To ensure the 
							application does not freeze a timeout is used.
					send(a_string: STRING)
							After connect succeeded this feature can be used to send a string to the other peer. This is done by putting
							the packet to send into a send_queue from where the UDP_SEND_THREAD takes it out and sends it to the other peer.
					receive_blocking:STRING
							Waits until there is a string put into the receive_queue by the UDP_RECEIVE_THREAD and then returns the string
					receive:STRING
							Like receive_blocking:STRING
					receive_non_blocking:STRING
							Returns the top of the receive_queue if there is something otherwise returns Void. But never waits (non_blocking)
					manager_terminated: BOOLEAN
							Returns whether stop was called.

				The following fields tell the type of error that occurred after having called the corresponding feature from above

					register_error_type: INTEGER_64
					unregister_error_type: INTEGER_64
					connect_error_type: INTEGER_64
					registered_users_error_type: INTEGER_64	

				The following features are not public and should not be accessed

					parse_packet(packet: PACKET): detachable JSON_OBJECT
							PACKET only provides access character by character. Therefore we loop over the packet and build the 
							string that represents the JSON_OBJECT. After that we parse the string to an object.
							This method is exclusively used in UDP_RECEIVE_THREAD after having received a packet.
					process(json_object: JSON_OBJECT)	
							This method is also used exclusively in UDP_RECEIVE_THREAD after having successfully called parse_packet.
							Likewise on the server side we first look at the type of the packet. According to the type a corresponding
							handler is called.

	TARGET_PACKET:		This class is basically a PACKET with an additional field peer_address which is used by the UDP_SENDER_THREAD
				so that it knows who to send the message to. Furthermore it has some helper features that ease the creation of a packet
				such as fill. For every message_type there exists an appropriate creation feature so that in the CONNECTION_MANAGER
				only create packet.make_register_packet("Anna"); send_queue.extend(packet) has to be called.

	MUTEX_LINKED_QUEUE:	When sending a packet CONNECTION_MANAGER puts it in a MUTEX_LINKED_QUEUE called send_queue. On the other side UDP_RECEIVE_THREAD
				periodically checks whether there is something in the queue and if so sends the packet. 
				When receiving a packet in UDP_RECEIVE_THREAD and if the packet is for the user it is pushed into a MUTEX_LINKED_QUEUE 
				from where a client can read it in CONNECTION_MANAGER receive. 
				This architecture allows to separate the client application (main thread) from receiving and sending. As multiple Threads
				access the queues the access is only given while holding a lock on a MUTEX.
	
	UDP_SEND_THREAD:	A Thread that periodically checks whether there is a TARGE_PACKET in the send_queue and if so sends it to the address given in the packet.
	
	UDP_RECEIVE_THREAD:	A Thread that listens on the socket for incoming packets. After having received a packet it uses the CONNECTION_MANAGER's parse_packet and
				process. Therefore it is client of CONNECTION_MANAGER.
	
	KEEP_ALIVE_THREAD:	This Thread is launched when a connect in CONNECTION_MANAGER succeeded. It periodically sends keep-alive packets to the other peer.

	UTILS:			This static class provides the constant that are necessary for the p2p protocol to run. Therefore a lot must be 
				equal to the UTILS class of the Rendezvous_Server. For example the error_type constants. Additionally there are constants
				like server_ip or server_port that must be adjusted according to the server. Also the different timeouts and intervals might be 
				changed according to the given network architecture. When setting debugging to true, the outputs from UDP_SEND_THREAD and 
				UDP_RECEIVE_THREAD are displayed

### 5. Step-by-Step Guide
---------------------
1) As a first step you need to download the resources from our bitbucket side:

Using git: `git clone git@github.com:silvanegli/Eiffel-P2P.git`
There are two folders: "Client_Interface" and "Rendezvous_Server"

2) Setting up the Server:

It's important to make sure that the server is available from everywhere you want to acces it.
If you want to access the server from the internet, but you install the Server in your LAN, you may have to configure your router.
So make sure that your server is available.
  
Open the Project "Rendezvous_Server" in Eiffel Studio, you may finalize and run the project.
Now it's important to get the Server's IP, this can either be done by terminal ("ipconfig" windows, "ifconfig" linux) or if you using a cloud server, then the provider should give you the address.

3) Setting up the Client:

I assume that you will use this Interface in a program.
As a first step, you have to copy the Client_Interface folder into the root of your Eiffel project.
Then restart EiffelStudio, clean and compile your project.
The Client Interface should now appear in your project structure.

3.1) Setting up UTILS.e

Open UTILS.e located in the root of Client_Interface.
Then change "server_ip" to your server ip from step (2).

If you need you can change the timeouts, intervals or the maximum_packet size.
This will depend on the use of your program.

Now save the project.

3.2) Initialize connection

In your project you first have to initialize an object of type CONNECTION_MANAGER (see (4) for more details).

	local
		man: CONNECTION_MANAGER
		...
	do
		...	
		create man.make
		...

Now initially you have to register yourself on the server, with a ID. (The Server will link this ID to your computer)

	success := man.register(id)

This function returns a boolean success. If success is true, then the connection is established.
Now you are successfully connected to the server

3.3) Find another client

To connect to an other client you have to get the peer's ID on the server.
You can get a list of all logged in clients with this feature:

	success := get_registered_users

If success is true then you'll find the registered users in:

	array := registered_users	

Choose the client a_peer to connect to from the array list.

3.4) Connect to another client

Assuming you want to connect to a_peer choosed in 3.3), you have to enter:

	success := man.connect (a_peer)

If connect returns true, then you are successfully connected to the remote client.

IMPORTANT: The client on the other side, has to do this quite at the same time (10 sec), in order to successfuly connect to each other.
This is because of the UDP_hole Punch. This is left to the USER to garantee this.

3.5) Sending/Receiving stuff

Now you are able to send data through the internet. To send data use:

	man.send (a:STRING)

You may call the Receive function in a separate Thread. For Receive:

	a_string := man.receive			// Equal to a_String := man.receive_blocking
	a_string := man.receive_non_blocking	// Is non blocking, if there is nothing to receive it returns Void
	a_string := man.receive_blocking	// BLocking, blocks until it can return the latest value.

4) Now you may run the project system.

### 6. Example
----------

For better understanding there is an implementation of a peer-to-peer chat which can be
found in [Client_Interface/examples](https://github.com/silvanegli/Eiffel-P2P/tree/master/Client_Interface/examples/Chat)

### 7. Future trends /issues 
------------------------

	Security:		Currently there is no authentication integrated an implementation with a user requiring a password to register or unregister 
				would be nice
	TCP:			This implementation is fully based on UDP. We tried to use TCP (and TCP Hole Punch) but had a lot of problems.
				A solution also supporting TCP would be nice regarding reliability, keep-alive overhead
	connect:		So far The CONNECTION_MANAGER.connect feature must be called on both sides within the
				given connecting_timeout. A solution where one side waits for the other side to connect, likewise TCP listen/connect
				would also be possible.
	Network Architecture:	For two devices sitting behind the same NAT device (being in the same network) a nice extension would be to also
				check whether a connection with private IP/Port succeeds and if not going on with this code. Therefore also the private IP/Port 
				would have to be stored on the server.
	NAT:			There are many different kinds of NAT and especially for Symetric NAT UDP Hole Punching can have problems.
	Platform: 		The system was tested for Linux and Windows Microsoft but not for Mac OS


### 8. Sources
----------

http://www.bford.info/pub/net/p2pnat/index.html

