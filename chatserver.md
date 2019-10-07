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
 ‏‏‎ 
 ‏‏‎ 
  ‏‏‎ 
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

```handle_cmd``` function is the heavy-lifter. As the name implies, it handles commands. Just like everything else, ```handle_cmd``` function’s routine is really elegant and simple.

In case of a command that only requires the server to send a message to a particular client, it returns False otherwise it returns a message.

```handle_cmd``` takes the sender’s socket and data as arguement where data is a dictionary and NOT a message.

``` def handle_cmd(sck, data): ```

We get the command by simple splitting and slicing, we get the sender’s addr to check if they are OP and their nickname.

```
cmd = data.get("msg")[1:]
addr = sck.getpeername()
nick = data.get("nick")
```

Then we do rudimentary parsing to check the command’s name. If the command’s name does not pass any of the commands, we simply return the data.

If the command is  ```ban```, then we need to check if the  ```addr``` is OP. If not, then we warn the user.

```
if cmd.startswith("ban"):
    if addr not in OP:
        msg = gen_msg("you dont have permission to use this command")
        ok  = send_only(msg, sck)
        return False
    return ban(cmd.split()[1:])
```

If the command is ```disconnect```, remove the ```addr``` from ```SOCKETS``` and  ```NICKS```.

```
elif cmd.startswith("disconnect"):
    SOCKETS.remove(addr)
    del NICKS[nick]
    return gen_msg(f"{nick} disconnected")
```    
    
If the command is  ```nick```, we change their old nickname to their new one. To achieve this, we delete their entry in  ```NICKS``` and replace it with a new one.

```
elif cmd.startswith("nick"):
    new_nick = cmd.split()[1:]
    del NICKS[nick]
    NICKS[new_nick] = addr
    return gen_msg(f"{nick} is now {new_nick}")
else:
    return data
```    
If you noticed, we have two undeclared functions ```ban``` and ```list_users```. Since these commands need a little extra work, they have separate helper functions.

In the following sections, we implement them.

# Ban

Whenever a community is created, there’s always trolls and other pesky people who ruin everyone’s experience. The classic method of handling these people is simply barring them from using the service. In our protocol, we call it ban and it alters the following data structures.

- ```NICKS```
- ```SOCKETS```
- ```BANNED```
Here’s the routine for the  ```ban``` function:

We check if the user that ran the command is OP.
We check if the targetted user is already banned.
We remove the targetted user from  ```NICKS``` and  ```SOCKETS``` .
We add them to  ```BANNED``` list.
We generate a message that lists out all the users that have been banned.
Step #1 is already checked in  ```handle_cmd```, so we need not worry about that.

In actual practice, we do step #5 alongside other routines to increase the efficiency.

The ban function takes a list of all the client’s nickname that are to be banned. We then set the number of clients to a variable called  ```nc``` and create another variable  ```msg``` to store the message temporarily.

```
def ban(clients):
    msg = "banned "
    nc = len(clients) - 1
```    
We loop through each client in  ```clients``` and get their  ```addr```.

```
for n, i in enumerate(clients):
    addr = NICKS.get(i)
```    
We check if they are in  ```BANNED``` list already and skip the iteration if they are.

```
if addr in BANNED:
    continue
```    
We append them to the  ```BANNED``` list and remove them from  ```SOCKETS``` list and the  ```NICKS``` dictionary.

```
BANNED.append(addr)
SOCKETS.remove(addr)
del NICKS[i]
```

We then add the nickname to  ```msg```.

```
msg += i
if n != nc:
    msg += " "
```

At last we return the message.

```
return gen_msg(msg)
```

# list

Another nifty command to have in hand would be the  ```list``` command. Simply put, it lists all the users online. The implementation is really simple; we loop through  ```NICKS``` and add each user to a variable, generate a message and return to sender.

```
def list_users(send_to):
    msg = "active users\n"
    nc  = len(NICKS) - 1
    for n, i in NICKS:
        msg += i
        if n != nc:
            msg += " "
    send_only(gen_msg(msg), send_to)
```

# Main loop

The main loop constructs a working server with all the building blocks that are functions. To get the main loop working, we need to initialize the  ```SERV_SOCKET``` among other things. We do these in the  ```init``` function.

The  ```socket``` module has two functions that are particularly useful when we initialize the server. Them being  ```gethostname``` and  ```gethostbyname```. As the name suggests,  ```gethostname``` returns the actual server’s (the hardware) hostname and  ```gethostbyname``` returns the IP address pointing to the hostname. We can use this to get the IP address of the server.

```
def init():
    host = socket.gethostbyname(socket.gethostname())
    saddr = (host, 9600)
    SERV_SOCKET.bind(saddr)
    SERV_SOCKET.listen()
    SOCKETS.append(SERV_SOCKET)
    NICKS["server"] = saddr
```

To make OPs and banned users permanent, we save them to files and read them once the server starts. To make thing easier, they are saved in  ```op``` and  ```banned``` respectively.

We read the aforementioned files and set  ```OP``` and  ```BANNED``` accordingly. Note that we have to read the files only if they are present.

Since  ```OP``` and  ```BANNED``` are list of  ```addr```, the way the data will be written will be the following:

```
IP1 port1
IP2 port2
```

To parse this file, we will create a small function inside  ```init``` that takes file path as an arguement and returns a list of  ```addr```. Then using that function, we set  ```OP``` and  ```BANNED```.

```
def read_file(path):
    res = []
    if os.path.isfile(path):
        with open(path) as f:
            for i in f.read().split("\n"):
                a, p = i.split()
                p = int(p)
                res.append((a, p))
    return res

OP     = read_file("op")
BANNED = read_file("banned")
```

To close the server cleanly, we need to use the close method of a  ```socket``` object. And we need to write  ```OP``` and  ```BANNED``` to files in the format specified before. To do these, we write a function  ```on_kill``` that performs the aforementioned actions.

```
def on_kill():
    SERV_SOCKET.close()
    def write_file(path, lst):
        with open(path, 'w') as f:
            for i in lst:
                f.write(f"{i[0]} {i[1]}\n")
    write_file("op", OP)
    write_file("banned", BANNED)
```

The main loop is simple in principle and it does very little on its own. This simplicity is reflected on the main loop’s routine.

We try to receive data from all readable clients. This is achieved by using the select function from the  ```select``` module. When given a list of file descriptors, it can return back a list of descriptors that are can be read. ```select``` function can also be used to check for writable and executable file descriptors!

The fourth arguement is timeout and we set it to zero because waiting for a client is pointless.

```
def main():
    readable, _, _ = select(SOCKETS, [], [], 0)
    for sck in readable:
```

If the socket that is to be read is  ```SERV_SOCKET```, then it implies that a new client is connected. We accept the  ```socket``` and get its  ```addr```.

```
if sck == SERV_SOCKET:
    sockfd, addr = SERV_SOCKET.accept()
```

If it’s a banned client, we ignore it. Send a nice message to banned client about their state.

```
if addr in BANNED:
    send_only(gen_msg("you are banned"), sockfd)
```

Otherwise we add it to  ```NICKS``` and  ```SOCKETS``` and we send a join message to everyone.

Just after the request to join, we expect the client to send its nickname as a raw string (the data sent would be “john”).

```
else:
    nick = sockfd.recv(4096)
    send_all(gen_msg(f"{nick.decode()} connected"),
             SERV_SOCKET)
    SOCKETS.append(sockfd)
    NICKS[nick] = addr
```

If its any other socket, then it means they sent data. So we try to read them.

```
else:
    try:
        data = sck.recv(4096)
```

We parse it to see if it’s a command. For a message to qualify as a command, it needs to start with a  ```/```. If it is a command, we let  ```handle_cmd``` do its work.

```
data_dict = json.loads(data)
if data_dict.get("msg").startswith("/"):
    data = handle_cmd(sck, data_dict)
```

If  ```handle_cmd``` returns False, then we skip the iteration because we need not send any message to everyone. Otherwise, we simply send the data to everyone.

```
    if not data:
        continue
send_all(data, sck)
```

If the socket is broken, for whatever reason, then it raises  ```ConnectionResetError``` error. This happens mostly when the client decided to disconnect. So we remove the broken socket from the data structures. And then send a disconnect message.

```
except ConnectionResetError:
    addr = sck.getpeername()
    nick = ""
    for k, v in NICKS.items():
        if v == addr:
            nick = k
    if nick:
        send_all(gen_msg(f"{nick.decode()} disconnected"),
                 SERV_SOCKET)
        del NICKS[nick]
    SOCKETS.remove(sck)
```

We only want the main loop to be run when the server is run from the terminal. We use the  ```__name__``` special variable to achieve that.

If an administrator wishes to kill the server, they can do by pressing  ```^C```. When they do that,  ```KeyboardInterrupt``` is raised.

```
if __name__ == "__main__":
    init()
    while 1:
        try:
            main()
        except KeyboardInterrupt:
            break
    on_kill()
```

# Client

The client has two important parts - UI and customization. The UI is made using a classic library -  ```curses```. Curses  originally made for Unix terminals is a TUI library that can be used to make complex TUI with relative ease. One can run a curses program in NT using libraries like PDCurses. Pypi has  ```windows-curses``` that installs the library in the correct place.

Customization, through commands, makes the client a lot flexible than one might think at first. One can even write their commands in any language and use their output using modules like  ```subprocess```; although this might a bit inefficient but it does allow some kind of flexibility. In some cases, it might be even faster than an implementation done purely in Python.

Implementing a custom command is really easy. All one needs to do is make a function that takes a dictionary as an arguement and return a value.

If the return value is  ```False```, then it implies that a message should not be sent to the server. For example, take  ```ping```,  ```pong!``` is sent as a reply to the user but we need not to send anything back to the server, so we return  ```False``` in  ```ping```’s implementation.

If the return value is a dictionary, then the dictionary, as JSON, is sent to the server. For example, take  ```switch-case```, when given a message it swaps the case of the message and it returns a dictionary whose  ```msg``` value’s case is swapped. This, then, is sent to the server.

Since commands have access to the UI and the message, they can pretty do much anything meaning infinite customizability!

# UI

We will save the UI in a file named  ```ui.py```.

The UI is made using  ```curses```.

```
import curses
import json
```

```
from select import select
```

Then we make a UI class that has variables that determines the prompt and format of the printed message. We store these in  ```prompt``` and  ```fmt``` respectively.

We also need the socket object of the client so we can read data from the server. We need the nickname of the user so we can print the message.

```
class Ui:
    def __init__(self, socket, nick):
        self.prompt = "> "
        self.fmt    = "{nick} - {msg}"
        self.socket = socket
        self.nick   = nick
```

We set the default value of  ```prompt``` to be an arrow and of  ```fmt``` to be  ```{nick} - {msg}```.

When one prints a message to the screen,  ```{nick}``` changes to the sender’s nickname and  ```msg``` changes to the sender’s message. How do we go about doing that? Well its simple because of  ```str```’s method called  ```format```. When given a dictionary,   ```{key}``` changes to corresponding value.

Suppose a message is like this

```
{
    "nick": "john",
    "msg": "hello guys!"
}
```

when printed, it changes to  ```john - hello guys!```.

We have to make two curses windows for showing messages and taking inputs.


Input is taken from a tiny box at the bottom of the screen and the messages are printed above. The width of the message window is 100% but the height is one less than the height of the screen.

The width of the input window is 100% yet again but the height is one.

To get the maximum height and width (called lines and columns respectively), we create a stdscr object.

```
stdscr = curses.initscr()
lines, columns = stdscr.getmaxyx()
```

Then we make the input window and the input window with the dimensions mentioned above.

```
self.winput = curses.newwin(1, columns, lines - 1, 0)
self.wmsg = curses.newwin(lines - 1, columns, 0, 0)
```

Then we set the message window to be scrollable and the input window to have nodelay i.e. there will no delay between the user keypress and the buffer the input is stored in.

```
self.winput.nodelay(True)
self.wmsg.scrollok(True)
```

Since we do not want the user to be printed in the main window and want the user input to be available as soon as possible, we do the following.

```
curses.noecho()
curses.cbreak()
```

We will draw the prompt after starting the UI.

```
self.winput.addstr(self.prompt)
```

We will define a method called  ```__print__``` that takes a message, in JSON, and prints it to the message window.

```
def __print__(self, data):
    data = json.loads(data)
    self.wmsg.addstr(self.fmt.format(**data))
    self.wmsg.addstr("\n")
    self.wmsg.refresh()
```

We will create a function that runs forever and is indented to be run in a separate. This function receives data from the socket and prints it to the message window. We will call this function  ```do_msg```.

```
def do_msg(self):
    while 1:
        readable, _, _ = select([self.socket], [], [], 0)
        if readable:
            data = self.socket.recv(4096)
            self.__print__(data)
```

We will create a function which creates a message encoded in JSON when given a raw string. This is useful in the input routine. We call this function  ```__mkdata__```.

```
def __mkdata__(self, msg):
    data = json.dumps({"nick": self.nick, "msg": msg})
    return bytes(data, "utf-8")
```

```do_input``` is a function that is intended to run in the main thread and it takes input from the user and returns it as a string.

It returns 1 when the main loop should not send any data to the server.

It has  ```inp``` as an arguement which acts as a buffer for the message.

```
def do_input(self, inp):
```
We get the current cursor position and the current character in the buffer. We will also set a variable named  ```can_send``` that tells the main loop if the data has to be sent to the server. We will set it to  ```False``` at first but change it when the user presses enter.

```
self.can_send = False
cury, curx = self.winput.getyx()
ch = self.winput.getch()
if ch != curses.ERR:
```

If the  ```ch``` is newline, then the user pressed enter. We print the message and we refresh the window. We return 2 if  ```inp``` is empty.

```
if ch == ord("\n"):
    if not inp:
        return 1
    data = self.__mkdata__(inp)
    self.__print__(data)
    self.winput.clear()
    self.winput.addstr(self.prompt)
    self.winput.refresh()
    inp = ""
    self.can_send = True
```

If the user pressed backspace, we want to delete the current character.

```
elif ch in [curses.KEY_BACKSPACE, ord("\b"), ord("\x7f")]:
    self.winput.delch(0, curx - 1)
    inp = inp[:-1]
```

Otherwise we simply add the character to  ```inp```.

```
else:
    self.winput.addch(ch)
    self.winput.refresh()
    inp += chr(ch)
```

At last we return  ```inp```.

```
return inp
```

When the user wants to quit the client, curses has to be closed. So we will create a function  ```kill``` that safely closes curses.

```
def kill(self):
    curses.endwin()
```

# Commands

Before implementing client-side commands, we need a nice way to represent user’s details. To do this, we will create a dataclass which simplifies the class declaration a lot.

We will save this in the main file -  ```client.py```

To make a dataclass, we need to import  ```dataclass```

```
import socket

from dataclasses import dataclass
```

```User``` class will have three variables -  ```nick```,  ```socket``` and  ```server_addr```

It will also have a method -  ```connect```.  ```connect``` will be used to connect to a server that corresponds to the  ```server_addr```.

```
@dataclass
class User:
    nick: str
    server_addr: (str, int)
    socket: socket.socket
```

To connect to a server, we use the  ```connect``` method of a socket object.

```
def connect(self):
    self.socket.connect(self.server_addr)
    self.socket.send(bytes(self.nick, "utf-8"))
```

Out of all the server-side commands,  ```nick``` and  ```disconnect``` needs some work the client-side as well. So we will implement these.

When implementing commands in  ```commands.py```, the end-user has access to two important classes -  ```UI``` and  ```USER```.

To implement  ```nick```, we simply change nick variable of  ```USER``` variable and we return back the dict without any changes.

```
import json

def change_nick(data):
    data_dict = json.loads(data)
    USER.nick = data_dict.get("nick")
    UI.nick   = USER.nick
    return data
```

To implement the  ```disconnect```, we need to  ```kill```  `UI``` and close  ```USER```’s  ```socket```.

```
def disconnect(data):
    USER.socket.send(data)
    USER.socket.close()
    UI.kill()
    return False
```   
We will implement the  ```connect``` command which connects to the given server. When the port is not defined, we will let it default to 9600.

```
def connect(data):
    dat = json.dumps({"nick": "client", "msg": "connected!"})
    data_dict = json.loads(data)
    addr = data_dict.get("msg").split()[1]
    addr = addr.split(":")
    if len(addr) == 1:
        port = 9600
    else:
        port = addr[1]
    USER.serv_addr = (addr[0], port)
    UI.__print__(dat)
    return False
```

To add some more customizability to the UI, we will add two commands -  ```prompt``` and  ```fmt``` which can change  ```UI```’s  ```prompt``` and  ```fmt``` variable respectively.

To implement these commands, we will split the msg field of data and change the corresponding value. Since we do not want to send any of this to the server, we will return False.

```
def prompt(data):
    msg = json.loads(data).get("msg")
    new_prompt = msg[len("/prompt "):]
    UI.prompt = new_prompt
    return False

def fmt(data):
    msg = json.loads(data).get("msg")
    new_fmt = msg[len("/fmt "):]
    UI.fmt = new_fmt
    return False
```

As an example, we will implement the  ```ping``` and  ```switch-case``` commands.

The  ```ping``` command simply needs to print  ```pong!``` back to the message window.

```
def ping(data):
    dat = json.dumps({"nick": "client", "msg": "pong!"})
    UI.__print__(data)
    return False
```

The  ```switch-case``` command needs to get the actual message by slicing the  ```msg``` key.

```
def switch_case(data):
    data = json.loads(data)
    msg = data.get("msg")[len("/switch-case "):]
    return UI.__mkdata__(msg.swapcase())
```

To let the client know which function should a command run, we will define a dictionary  ```CMD``` whose key is the command’s name and the value is the function.

If one desires to add more, they can do so by writing a function and adding a key:value pair to  ```CMD```.

```
CMD = { "nick": change_nick,      "ping": ping,
        "disconnect": disconnect, "switch-case": switch_case,
        "connect": connect, "fmt": fmt, "prompt": prompt}
```

# Main loop

We will get the initial server’s addr as the first command-line arguement. Since we are initliazing the client, we put these in the init function.

Using the sys module, one can get access to the command-line arguements via the argv variable. It is a list of all command-line arguments and its zeroth element is the path to the file that is currently running. If the client is not started with any arguements, we will simply quit with a message.

```
def init():
    import sys
    if len(sys.argv) == 1:
        print("server ip not provided!")
        exit(1)
```

The server  ```addr``` will provided in the format -  ```ip:port```. If  ```port``` is not defined, then we will default to 9600.

We will also create a socket object.

```
ad = sys.argv[1].split(":")
if len(ad) == 1:
    port = 9600
else:
    port = int(ad[1])
saddr = (ad[0], port)
sck = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
```

We will now create instances of  ```Ui``` and  ```User``` classes and make it global.  ```USER``` environmental variable will be used as the default nickname for the client. To get an environmental variable, we will use the  ```getenv``` function in the  ```os``` module.

```
global USER, UI
import os
import ui
nick = os.getenv("USER")
USER = User(nick, saddr, sck)
UI   = ui.Ui(sck, nick)
```

Then we will connect to the server.

```
USER.connect()
```

To get the actual client working, we will define a  ```main``` function that creates a thread for the message window.

```
def main():
    msg_thread = threading.Thread(target=UI.do_msg)
    msg_thread.start()
```

Then we will start the input window which will run in the foreground.

```
inp = ""
while 1:
    inp = UI.do_input(inp)
    if inp == 1:
        inp = ""
        continue
```

If  ```UI.can_send``` is  ```True```, then we can send data to the server. But before that we will have to check if it’s a command and evaluate it.

Here,  ```inp[1:]``` will be the command’s name. We check if it is in  ```CMD```, then run the corresponding the command. Each command requires a message object, so we give data as an arguement.

Then we send the data to the server if  ```data``` is not  ```False```.

```
if UI.can_send:
    data = UI.__mkdata__(inp)
    if inp.startswith("/"):
        if inp[1:] in CMD:
            data = CMD[inp[1:]](data)
    if data:
        USER.socket.send(data)
```

If the special variable  ```__name__``` is  ```__main__```, then we will run the client.

```
if __name__ == "__main__":
    init()
    from commands import *
    import threading
    main()
```

# Bibliography

1. http://www.literateprogramming.com
2. https://en.wikipedia.org/wiki/Literate_programming
3. https://en.wikipedia.org/wiki/Unix_philosophy
4. https://en.wikipedia.org/wiki/Curses_(programming_library)
5. https://docs.python.org/3/library/dataclasses.html
6. https://www.youtube.com/watch?v=T-TwcmT6Rcw
