# nginx-naxsi

##Securing your app with Nginx Naxsi

After experimenting with Nginx, I decided to look into the available security modules. One module I seen mentioned a lot was Naxsi – a web application firewall for Nginx.

First off, you will need to install Naxsi. I’m working on Ubuntu 16.04 LTS and make the following shell:

https://gist.github.com/dcnl1980/eaba5da5abf91d9cb76490c54d32880c

After the installation is complete, you must edit the /etc/nginx/nginx.conf configuration file and add the following line into the http code block. Your conf file should look something like the below:


http {
    ## NAXSI Security
    include /etc/nginx/naxsi_core.rules;
    ...
    
The naxsi_core.rules you can find in:

https://gist.github.com/dcnl1980/a82cd9ad5805e8978660e47f3fcfbbd4
   
Next stage is to create a custom rules file in /etc/nginx/. This can be called what ever you like – in this example I have called mine ga-naxsi.rules. Copy and paste the below code into this new file:

LearningMode; #Enables learning mode
 SecRulesEnabled;
 #SecRulesDisabled;
 DeniedUrl "/RequestDenied";
 ## check rules
 CheckRule "$SQL >= 8" BLOCK;
 CheckRule "$RFI >= 8" BLOCK;
 CheckRule "$TRAVERSAL >= 4" BLOCK;
 CheckRule "$EVADE >= 4" BLOCK;
 CheckRule "$XSS >= 8" BLOCK;
Note that Naxsi is in LearningMode above. This means that malicious requests are copied to the a defined error log and not blocked. This is useful for creating the whitelist later on in this article.

Next stage is to edit the sites virtual host configuration. My file is called default and is located in /etc/nginx/sites-enabled. In the “location” code block add the following include, set the RequestDenied path and ensure your error log is defined:

location / {
   # NAXSI Security
   include "/etc/nginx/ga-naxsi.rules";
   ...
}
 
location /RequestDenied {
    return 418;
}
 
error_log /var/log/nginx/ga_error.log;
access_log /var/log/nginx/ga_access.log;
Now all the main configuration is done, its time to restart Nginx by typing the following command:

sudo service nginx restart

###Learning Mode
As previously mentioned, we have set Naxsi to learning mode. This allows Naxsi to provide you with white lists based on users activity. For example, a malicious request is made but not blocked – instead it is copied to /var/log/nginx/ga_error.log.

Lets say I have a form on my website, and I post some malicious content:

naxi-form

Now if I look in /var/log/nginx/ga_error.log, I should see a logged malicious request. If you see something like the below, then Naxsi is working in Learning mode!


2014/04/21 20:41:59 [error] 31267#0: *1 NAXSI_FMT: ip=*.*.*.152&server=*.*.*.*&uri=/&learning=1&total_processed=1&total_blocked=1&zone0=ARGS&id0=1011&var_name0=firstname&zone1=ARGS&id1=1302&var_name1=firstname&zone2=ARGS&id2=1303&var_name2=firstname&zone3=ARGS&id3=1308&var_name3=firstname&zone4=ARGS&id4=1309&var_name4=firstname&zone5=ARGS&id5=1313&var_name5=firstname, client: *.*.*.*, server: *.*.*.*, request: "GET /?firstname=%3Cscript%3Ealert%280%29%3B%3C%2Fscript%3E&lastname= HTTP/1.1", host: "*.*.*.*", referrer: "http://example.com?firstname=test&lastname="

##Creating the whitelist

Once you are happy with the amount of user activity and are ready to switch learning mode off, you will need to download nx_util. You can do this by running the following command (I’m in /root/):

apt-get install python
wget https://naxsi.googlecode.com/files/nx_util-1.0.tgz
tar -zxf nx_util-1.0.tgz
cd nx_util-1.0/nx_util

Run the following command to get optimised rule suggestions:

python nx_util.py -c nx_util.conf -l /var/log/nginx/* -o
This should output something like the below. Copy this into your custom rule file (e.g /etc/nginx/ga_naxsi.rules) and comment out learning mode from LearningMode; to #LearningMode;.

########### Optimized Rules Suggestion ##################
total_count:1 (50.0%), peer_count:1 (100.0%) | parenthesis, probable sql/xss
BasicRule wl:1010 "mz:$URL:/|$ARGS_VAR:firstname";
total_count:1 (50.0%), peer_count:1 (100.0%) | ; in stuff
BasicRule wl:1008 "mz:$URL:/|$ARGS_VAR:firstname";

To enable the changes, restart Nginx

sudo service nginx restart
Now if I try and post malicious content through the form again (same as the above) – Naxsi should block this request and return a 418 status code (teapot).

If you wish to set a custom error page, you can so so by simply adding the below line to your virtual hosts config file:

error_page 418 /418.html;
