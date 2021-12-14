# Installing, configuring, and running `ucspi-ssl sslserver`

### What is `ucspi-ssl sslserver`?
`sslserver` is part of the [ucspi-ssl](https://www.fehcom.de/ipnet/ucspi-ssl.html) collection of tools used for building client-server applications that connect over SSL/TLS.
It was developed as an SSL/TLS analogue to Daniel Bernstein's [ucspi-tcp tcpserver](https://cr.yp.to/ucspi-tcp.html) program, 
and has been developed and maintained over the years by [William Baxter](https://github.com/SuperScript/ucspi-ssl), [Scott Gifford](https://github.com/scottgifford/ucspi-ssl), and [Erwin Hoffmann](https://www.fehcom.de/ipnet/ucspi-ssl.html).
The project's current home is at Erwin Hoffmann's site at [https://www.fehcom.de/ipnet/ucspi-ssl.html](https://www.fehcom.de/ipnet/ucspi-ssl.html).

`sslserver` listens for incoming TCP connections on a designated port.  Upon receiving an incoming connection from a client, `sslserver` negotiates a SSL/TLS session with the client, then spawns a child program that interacts with the client.  Requests sent from the client are piped as input to the child program through STDIN, and output written by the child program to STDOUT are piped as a response back to the client.  `sslserver` is capable of handling multiple simultaneous connections, as it will spawn a separate instance of the child program for each incoming connection from a client.

For more information, see [https://www.fehcom.de/ipnet/ucspi-ssl.html](https://www.fehcom.de/ipnet/ucspi-ssl.html).

### Getting `ucspi-ssl sslserver` up and running
It took me a fair amount of time (and Googling, and trial-and-error, and even some help from Erwin Hoffman) to get `ucspi-ssl sslserver` up and running, as there are a number of nuances in the process.  So, I thought I would document the steps that worked for me to get `ucspi-ssl sslserver` up and running to have as a reference for myself, as well as for others that may find this useful.

For this example, we'll use `ucspi-ssl sslserver` to create a simple SSL/TLS web server that runs on port 443, which uses a short python script to create a dynamic web page.  I used a VPS server running Debian 11 x64 from Digital Ocean for this example.

##### Step 1
Spin-up a VPS running Debian 11 x64.  I used a 'basic droplet' from Digital Ocean with 1GB of RAM and an Intel vCPU.

##### Step 2
Create a FQDN (e.g. example.domain.tld) and point it to the IP address of the VPS.

##### Step 3
Login to the server as root.

##### Step 4
Update the server:

    apt-get update
    apt-get dist-upgrade
    
##### Step 5
Install tools and dependencies that will be needed for the build:

    apt-get install build-essential
    apt-get install libssl-dev
    
##### Step 6
Install fehQlibs-19:

    cd /usr/local/
    wget https://www.fehcom.de/ipnet/fehQlibs/fehQlibs-19.tgz        #fetch the source tarball from www.fehcom.de
    md5sum fehQlibs-19.tgz                                           #verify the integrity of the download.  should be 9ab8703dfc510958fb6befa822ed7bad
    tar -zxvf fehQlibs-19.tgz                                        #extract the archive 
    cd fehQlibs-19/
    make                                                             #compile
    cd ..
    ln -s fehQlibs-19 qlibs                                          #create a symlink so that ucspi-ssl can find fehQlibs-19
    
##### Step 7
Install ucspi-ssl-0.12.3:

    cd /
    mkdir /package
    cd /package/
    wget https://www.fehcom.de/ipnet/ucspi-ssl/ucspi-ssl-0.12.3.tgz  #fetch the source tarball from www.fehcom.de
    md5sum ucspi-ssl-0.12.3.tgz                                      #verify the integrity of the download.  should be c1da77ee060ab79d37e1e73245d69a5a
    tar -zxvf ucspi-ssl-0.12.3.tgz                                   #extract the archive
    cd /package/host/superscript.com/net/ucspi-ssl-0.12.3
    package/install                                                  #compile

##### Step 8
Create a DH param file:

    cd
    openssl dhparam -out /etc/ssl/dh2048.pem 2048
    
##### Step 8
Create an SSL certificate for the FQDN (e.g. example.domain.tld).  I'll use Let's Encrypt as the CA for this example.

    cd
    apt-get install certbot
    certbot certonly --standalone
    
certbot creates the private key and the certificate, and the certificate is signed by Let's Encrypt.  All of the necessary files are in /etc/letsencrypt/live/example.domain.tld/.
To be used with sslserver, the entire certificate chain and the private key must be bundled in a single file:

    cp /etc/letsencrypt/live/example.domain.tld/fullchain.pem /etc/letsencrypt/live/example.domain.tld/cert.pem
    cat /etc/letsencrypt/live/example.domain.tld/privkey.pem >> /etc/letsencrypt/live/example.domain.tld/cert.pem

Next, copy cert.pem to /etc/ssl/:

    cp /etc/letsencrypt/live/example.domain.tld/cert.pem /etc/ssl/
    
##### Step 9
Create a lower-privilege user to run sslserver under:

    adduser lowprivilegeuser
    
Proceed as per the prompts. 
    
##### Step 10
Login as lowprivilegeuser and copy the following test script (index.py) to /home/lowprivilegeuser/index.py:

    #!/usr/bin/env python3
    
    #for testing only, do not use for production.
    import sys
    import datetime
    
    requestheaders={}
    requeststring=None
    
    #get request headers
    for line in sys.stdin:
    	if(len(line.strip())==0): break
    	vals=line.strip().split(':')
    	if(requeststring is None and len(vals)==1): requeststring=line.strip()
    	if(len(vals)==2):
    		key=vals[0]
    		value=vals[1]
    		requestheaders[key]=value	
    	
    #send response headers
    print('HTTP/1.1 200')
    print('')
    
    #send response body
    print('<html>')
    print('<body>')
    print('<h1>200 OK</h1>')
    print(str(datetime.datetime.now()), '<br>')
    print('<br>')
    print('request string:', requeststring, '<br>')
    print('<br>')
    print('request headers:<br>')
    for r in requestheaders:
    	print(r, ':', requestheaders[r], '<br>')
    print('</body>')
    print('</html>')
    
    #flush
    print('', flush=True)

This python script receives a HTTP request from a client (e.g. web browser) through sslserver, then creates a dynamically-generated web page, and sends the response through sslserver to the client.
Note, this script does not include any input validation, provisions for handling timeouts, etc.; so it should not be used as-is for production, but should work fine with a standard web browser.

##### Step 11
Login is root, and copy the following control script (sslserverctl.sh) to /root/sslserverctl.sh:

    !/bin/sh
    start() {
    
      UID=`id -u lowprivilegeuser`
      GID=`id -g lowprivilegeuser`
    
      CERTFILE="/etc/ssl/cert.pem"
      export CERTFILE
      
      DHFILE="/etc/ssl/dh2048.pem"
      export DHFILE
    
      exec /usr/local/bin/sslserver -u $UID -g $GID -sH1 0.0.0.0 443 /home/lowprivilegeuser/index.py &
    
      echo "sslserver started."
    
    }
    stop() {
      pkill sslserver
      echo "sslserver stopped."
    }
    
    case "$1" in
      start)
            start
            ;;
      stop)
            stop
            ;;
      *)
            echo "Usage: $0 {start|stop|}"
            exit 1
    esac
    exit 0

Then, enable execution permissions for root:

    chmod u+x /root/sslserverctl.sh
    
This shell script is used to start sslserver so that it listens for incoming connections on port 443, negotiates an SSL/TLS session with the client using the certificate and DH param files created earlier, and spawns an instance of index.py for each incoming connection.

To start sslserver: `/root/sslserverctl.sh start`

To stop sslserver: `/root/sslserverctl.sh stop`

##### Step 12
Test.  Start sslserver:

    /root/sslserverctl.sh start

Now, point your web browser to https://example.domain.tld/.  If everything worked as it should, you should see a dynamically generated web page produced by index.py!

### Thank You
Special thanks to Erwin Hoffmann for his advice on how to get sslserver to see the dh param file.

### Questions?
[Contact MTI](https://www.meixler-tech.com/contact.php).







