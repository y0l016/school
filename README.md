# Introduction: 

This Investigatory project – The ChatServer is being presented in the form of literate programming. Literate programming is a programming paradigm in which the code is being explained in a way that we the humans could easily understand rather interpreting like a computer. We are meant to read, and so we’ve opted for literate programming so that one can easily understand the code when referring it in the future. This document can be tangled down into the program.  

This entire project is made using free and/or open-source software such as Emacs, CPython, Vim, etc. and licensed under BSD 2 Clause “Simplified” License. 

 

# Concordat: 

When beginning this project, we wanted the message data to be simple, elegant and portable. And so, the concordat was framed with reference to these aspects in mind. In order to comply with maximum portability, we’ve opted to use JSON, JavaScript Object Notation which is supported by most programming languages for the data struct part. 

The data structure of the message is composed of two key:value pairs. The keys to be named as  ```nick``` and ```msg```.  The  ```nick``` refers to the client's nickname (which will serve as the identifier on users' side) and and ```msg``` is the message the client wants to send.

The message data would look like this:

```
{
 "nick": "yolo",
 "msg": "hey spoon"
}
```
message would refer to the message's data structure hereupon.

In some cases the server would want to send some message to the users(like, 'Hey! you are banned') . In such cases the   ```nick``` is set to  ```server``` .

This message would be sent to clients online except the sender . The data is not modified by the server but in some cases the server can decide to ignore sending data which'll be discussed in forecoming sections.

# Commands

The Command feature provides a great potential of control over the chat being carried out in the chatserver. It is meant to enrich the chat experience.

The Commands are of two types, namely:

* Server-side commands
* Client-side commands

The antecedent one is to control the server side while the supervine is to modify/alter the client's message.

Server-side commands available are:

 * ```ban```
 * ```disconnect```
 * ```nick```
 * ```list```

Some of the above commands can only be executed by people with special appanages. These previlages can be held by an individual or a group of people. The Individual/Group of people holding these appanages are referred to as OP.  ```ban``` is one such OP command which can be executed only by OP/s.

Client side messages are accessible by all users and do not interact with the server directly in any way. Client side commands can be used to alter/modify the message.

Client-side commands implemented in this chatserver are:

* switch-case
* ping
* prompt

When a user tried the  ```ping``` command, the client would print  ```pong!``` .  The switch-case command would take the given string as an arguement and would swap it's case and the ```prompt``` command changes the ui's prompt. These are simple commands being implemented for now and many such commands can be implemented in the future as the command system basically exposes the whole client.

# Server

In the sever file we incclude all the necessary modules namely the  ```select``` , ```json``` and  the ```socket``` . The ```select``` function alone is being imported from the ```select``` module and is used for getting the list of active state sockets that are ready to be read.

```addr``` will refer to a tuple (host,port) where host is the ip of the client and port is the port to be connected to hereupon.

```
import json
import os
import socket

from select import select
```

- ```sockets``` is a list of all active(online) sockets including the server's socket.
- ```SERV_SOCKET``` is the socket object of the server.
- ```NICKS``` is a dictionary where the key is the nickname and value is the ```addr``` of the clients.
- ```BANNED``` is a list of all the banned clients.
- ```OP``` is a list of all ```addr``` that can perform administrative/OP actions.

```
SERV_SOCKET = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
NICKS       = {}
SOCKETS     = []
BANNED      = []
OP          = []
```

# Helper functions
Some code needs to be repeated many times inside the program. This situation might arise in two cases . For instance, everytime a message has to be recieved, the message recieving part of the code has to be repeated. This is one case. The other case is to handle errors. For example, when an online client goes offline, the server won't be ale to deliver the message to that client. In this case, the message sending function rasies an exception. And an error handling function is required multiple times to handle this error. To satisfy this code repeatation requirement without actually repeating the code, we use helper functions.

```send-only``` is a helper function used to send the message. It's arguements are ```data```` and ```send_to``` . ```data``` is a byte encoded messagge and ```send_to``` is the reciever's socket object.

When the client is offline and one tries to interact with its socket, it raises an error. This function handles the error by removing the socket from the following data structures so we don’t repeatedly try to send or receive data from a broken socket.

*NICKS
*SOCKETS

To remove the broken socket from  ```NICKS```, we use the  ```getpeername``` method of a socket that returns an  ```addr```.

```
def send_only(data, send_to):
    try:
        send_to.send(data)
    except:
        data_dict = json.loads(data)
        del NICKS[data_dict.get("nick")]
        SOCKETS.remove(sck)
```

```send_only``` is convenient to have but the program becomes verbose if we have a ```for``` loop in the middle of the main loop. Separating an integral part such as this to a function also makes debugging easier. So we write a function  ```send_all``` that sends a message to everyone except the sender.

```send_all``` is constructed on top of  ```send_only```. ```send_only``` does the big job. ```send_all``` sends the message to every socket except the server’s and the sender’s.

```
def send_all(data, sent_from):
    pass_scks = [SERV_SOCKET, sent_from]
    for sck in SOCKETS:
        if sck not in pass_scks:
            print(f"sending to {sck}")
            send_only(data, sck)
```

# Commands

Commands are an important part that our protocol offers. It is powerful and easily expandable. Implementing new commands should be a matter of second. As mentioned before, there are some commands that are exclusive to a certain group of users. When one tries to use a command that they don’t have access to, we need to let them know that they can’t use this command. Morever, we don’t need to send their mistake to everyone. Creating a message naturally becomes a vital part.

Creation of a message needs to be done quite often so we throw it in a separate function called  ```gen_msg``` that takes an arguement called ```msg```

```
def gen_msg(msg):
    msg = { "nick": "server", "msg": msg }
    return bytes(json.dumps(msg), "utf-8")
```
