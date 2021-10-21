---
title: SuiteCRM RCE Log File Extension Bypass 2
layout: posts
---
### CVE is still pending…
This one will be a bit short, since severity/impact/video/etc is all identical to [my post on the previous SuiteCRM RCE](/CVE-2020-28320-SuiteCRM-RCE/). CVE is still pending...

Metasploit module: [suitecrm_log_file_rce.rb](https://github.com/rapid7/metasploit-framework/blob/master/modules/exploits/linux/http/suitecrm_log_file_rce.rb)

During remediation testing of [CVE-2020-28328](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-28328) I dug a bit deeper and found another bypass for file extensions that was so simple, I'm a bit embarrassed I didn't spot it on the first go.
## Technical details
So, once the previous fix was released, I obviously wanted to check out what they did.

[The Fix for CVE-2020-28328](https://github.com/salesagility/SuiteCRM/commit/1618af16eaa494c4551bac961e5ac8fc3d87ab8c#diff-e9704a2002d127cd455e1eb0507042080bb79d362091e770803ff69a31139d0f), which can be found in my previous post as well.  
```
if ($value === '') {
    $GLOBALS['log']->security("Log file extension can't be blank.");
    continue;
}
```
So, now the log file extension can't be blank. ezpz. Well... There is another little hole in the equation, though I'm sure some of the ninjas reading this post can probably find even more. 

I looked down a few lines and `config['upload_badext']` caught my eye on [line 86](https://github.com/salesagility/SuiteCRM/blob/1618af16eaa494c4551bac961e5ac8fc3d87ab8c/modules/Configurator/Configurator.php#L86).
```
$trim_value = preg_replace('/.*\.([^\.]+)$/', '\1', $value);
if (in_array($trim_value, $this->config['upload_badext'])) {
    $GLOBALS['log']->security("Invalid log file extension: trying to use invalid file extension '$value'.");
    continue;
}
```
I wondered what was in that array... So, I used the trusty search feature in GitHub and searched `upload_badext` and a few results down, I see a promising code snippet in [download_modules.php](https://github.com/salesagility/SuiteCRM/blob/d57e91389d97791fe621d811f03fe05f8f5a7f78/install/download_modules.php#L60) on line 60 under the `/install/` directory.
```
if (empty($sugar_config['upload_badext'])) {
    $sugar_config['upload_badext'] = array('php', 'php3', 'php4', 'php5', 'pl', 'cgi', 'py', 'asp', 'cfm', 'js', 'vbs', 'html', 'htm');
}
```
The key thing to note here is that the user input was never converted to lower-case before being compared to the values in that array. So, if you use something like `.pHp` for the `logger_file_ext` value, you can perform the exact same attack I outlined in [my previous post](/CVE-2020-28320-SuiteCRM-RCE/) (or if you just want the exploit, [EDB-49001](https://www.exploit-db.com/exploits/49001)). 

I know... I can't believe I didn't check for this before. I feel so silly...

[The new fix](https://github.com/salesagility/SuiteCRM/blob/9cb957e4f41562eb44f6ce8c982e2a3c169fc951/modules/Configurator/Configurator.php#L103)
```
$badext = array_map('strtolower', $this->config['upload_badext']);
if (in_array(strtolower($trim_value), $badext)) {
    $GLOBALS['log']->security("Invalid log file extension: trying to use invalid file extension '$value'.");
    continue;
}
```
So you can see now that they now coonvert all the "bad extensions" from the config to lowercase, as well as the incoming extension.

So anyways, another point goes to testing fixes. Even if they did fix the original issue, maybe you overlooked something super simple and you find another bug!

## Timeline
**[06 NOV 2020] :** Issue reported to security@suitecrm.com  
**[13 NOV 2020] :** Issue verified by SuiteCRM 
**[28 APR 2021] :** Version 7.11.19 released with fix  
**[18 MAY 2021] :** Email SuiteCRM to request status of CVE ID  
**[20 MAY 2021] :** This article published  
**[21 MAY 2021] :** Email SuiteCRM to request status of CVE ID  
**[21 MAY 2021] :** SuiteCRM replies CVE is still pending  
**[22 MAY 2021] :** Metasploit module submitted: [pull request](https://github.com/rapid7/metasploit-framework/pull/15231)  
**[03 JUN 2021] :** Metasploit module merged into `rapid7:master`: [commit](https://github.com/rapid7/metasploit-framework/commit/8b737c2c609fa72651c65b7705bccd6a988ffa1a)
