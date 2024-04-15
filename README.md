# Wireshark PCAP Analysis

## Objective

The goal of this network analysis lab is to practice using Wireshark to monitor and interpret packets to detect malicious activity.

### Skills Learned

- Getting familiar with how Wireshark works.
- Experience analyzing packets from a pcap file.
- Inspecting packets deeper by following HTTP/TCP streams.
- Learning how to use filters in to quickly find specific packets.
- Learning some of the CRUD operations, such as HTTP GET and POST

### Tools Used

- Wireshark for Packet Analysis
- PCAP file to load into Wireshark
- URL Decoder to read encoded data from HTTP streams

## Steps

### Step 1:
After opening up the PCAP in wireshark, the first thing I checked was the file properties to see the total duration of the capture from start to finish. In a real scenario, this is considered good practice to confirm that the pcap file you are looking at matches up with the one that set off an alert in a SOC enviornment.

![Wireshark Capture File Properties](/images/WiresharkProperties.png "Wireshark Capture File Properties")
*Fig 1: Wireshark Capture File Properties - This screenshot shows the details of a packet capture in Wireshark.*

### Step 2:
Next, I opened Statistics->Conversations, and selected the TCP tab to take a look at the TCP connections. I could see that there were a lot of connections coming from the source IP 10.251.96.4 to the destination IP 10.251.96.5 reaching out to multiple different destination ports. The source IP was also using the same source port when reaching out to different destination ports, which could be indicative of a port scan initiated by the source IP 10.251.96.4. The reason this is suspicious is because TCP usually uses a different unique source port when initiating new connections from a source to a destination.

![Wireshark TCP Connections](/images/WiresharkTCPConnections.png "Wireshark TCP Connections")
*Fig 2: Wireshark TCP Connections - This screenshot shows the details of the TCP connections initiated throughout this pcap file.*

### Step 3:
Looking through the beginning of the packet capture, I noticed that in packet 14, source host 172.20.10.2 initiated an HTTP connection with destination host 172.20.10.5. I right clicked on packet 14, then clicked Follow->TCP Stream. This opens up a new window that allows me to see more information about this connection. I was able to find out quite a bit from this connection:
    - Source: The user agent indicates that the requests are being made from a Windows 10 system using Chrome browser version 88.0.4324.146.
    - Destination: The destination address is a server that is running Apache 2.4.29 on Ubuntu. This information is disclosed in the server response headers.
    - The client is sending a GET request to the server to access various directories and files. The server responded with "HTTP 200 OK" for HTTP GET messages requesting access to "/" and "login.php", but responded with "404 Not Found Error" for "/favicon.ico". The 404 error indicates that the file may not exist on the server.

![Wireshark TCP Stream](/images/Packet14-TCPStream.png "Wireshark TCP Stream")
*Fig 3: This screenshot shows a TCP Stream of packet 14, emphasizing that the destination address is an Ubuntu Server.*

### Step 4:
Navigating further through the packet capture, packet 38 had a POST request sent from the Ubuntu Server address (172.20.10.5) to the client (172.20.10.2). This time, I opened up an HTTP Stream to gather more information about this connection. I was able to see login credentials with a username of "admin" and a password of "Admin%401234".

![Wireshark HTTP Stream](/images/Packet38-HTTPStream.png "Wireshark HTTP Sream")  
*Fig 4: This screenshot shows a HTTP Stream of packet 38, which shows a username and password.*

The "%40" in the password indicates that the actual character is encoded, so we will need to use a URL decoder in order to see the actual character. I went to https://www.urldecoder.io and typed in "%40". It returned the decoded character which is an "@" symbol. So, we are now able to put together that the password is "Admin@1234".

![URL Decoder](/images/URLDecoder.png "URL Decoder")
*Fig 5: This screenshot shows a URL Decoder website decoding "%40" into the "@" symbol.*

### Step 5:
I found another HTTP GET in packet 2215, and after looking at the HTTP stream I found something interesting. The user-agent in the HTTP GET request for the root directory was "gobuster/3.0.1". After doing some quick research into this, I found out that it is an open-source tool designed for brute forcing directories and files, DNS subdomains, and host names. Looking through the HTTP stream, I can tell that requests are being sent out to multiple directories and files, with a lot of them returning "404 Not Found Errors". I should be looking to see which "HTTP GET" requests from the client got an "HTTP 200 OK" response from the server. This is what I will want to investigate next.

![Wireshark HTTP Stream](/images/GoBuster.png "Wireshark HTTP Stream")
*Fig 6: This screenshot shows an HTTP stream of a GET request detailing the user-agent of the client initiating the request.*

### Step 6:
Since going through that HTTP Stream manually looking for "HTTP 200 OK" messages would take a lot of time, I can be more efficient by applying a filter within Wireshark. The two filters I want to apply in this case are the HTTP response code of 200 and the source IP address of server to narrow down our search. An easy way to do this is to find a packet that has an HTTP 200 response code. Once the packet is selected, I can navigate down and expand the application layer. Right click the HTTP Response Code->Apply as filter->Selected. The filter will automatically be applied to the search bar. 

Next, I want to find a packet that includes the server's IP address as the source. The process is very similar as before, however, this time, I will expand the layer 3 details of the packet. Right click the source IP address->Apply as Filter->&& Selected. This will add this filter to the one we specified earlier, refining our results even more.

![HTTP Response Filter](/images/HTTPResponseFilter.png "HTTP Response Filter")  
*Fig 7: This screenshot shows the process of applying a filter for a specific HTTP Response Code.*

This is what the filter looks like in the search after I have done the above process:

![Wireshark Filtered Search](/images/WiresharkFilteredSearch.png "Wireshark Filtered Search")  
*Fig 8: This screenshot shows the filter applied in the search bar.*

### Step 7:
Looking at the filtered search results, I inspected packet 7725 because it had a value that was much larger in the "Length" field than almost any other packet in the search results. After inspecting the HTTP stream once again, I scrolled down to find that the GET request to /info.php returned a lot of information. I made note of this and moved on to see if I could anything else that was interesting.

I found that the last GET request that gobuster/3.0.1 sent out seems to have been in packet 13661 attempting to read the /zn directory. After that, GET requests from the same IP were coming from a Linux OS using Mozilla Firefox

I continued on looking for any other packets that stood out, and I found a POST request. I followed the HTTP stream for this POST request and found some interesting information. The user-agent was sqlmap/1.4.7#stable (http://sqlmap.org) and I found some login credentials with the username "user" and password "pass". Doing a quick search, I was able to find out that the user-agent is another open-source tool designed to detect and exploit SQL Injection vulnarabilities. This is indicative of a malicious attempt to exploint SQL injection vulnerabilities and should be inspected further. 

![HTTP Post Request Stream Output](/images/POSTRequestHTTPStream.png "HTTP POST Request Stream Output")
*Fig 9: This screenshot shows the HTTP Stream of a POST request.*

### Step 8:
I searched through the packets looking to see if I could find any more POST requests that might indicate an SQL attack and I found an unusual POST request at packet 14060. As before, I followed the HTTP stream to gather more information.

![Abnormal POST Request](/images/AbnormalPOSTRequestHTTPStream.png "Abnormal POST Request")
*Fig 10: This screenshot shows the HTTP Stream of another suspicious POST request.*

I noticed that the data in the POST request is encoded, so I went back to https://www.urldecoder.io/ and copy/pasted the data in the POST request. This is what I found from the decoded data:

![Decoded POST Request](/images/POSTRequestDecodedOutput.png "Decoded POST Request")  
*Fig 11: This screenshot shows the decoded output from the data in the POST Request HTTP Stream*

This decoded output indicates that the components of this POST request contained a combination of SQL Injection, Cross-Site Scripting, and Command Injection and was trying to exploit all of these vulnerabilities in the same POST request. It looks like the command injection attack was used in an attempt to read the contents of the paswword file, which I can tell by this part: "('cat ../../../etc/passwd')".

### Step 9:  
I created a new filter to only output POSTs request from the attacker's IP 10.251.96.4 by typing in the search bar: "http.request.method == POST && ip.addr 10.251.96.4". This made it much easier to search throught the POST requests to find out more information.

![POSTFilter](/images/POSTFilter.png "POSTFilter")  
*Fig 12: This screenshot shows a filter being applied to only look for POST requests from 10.251.96.4.*

After aplying the filter, I found another interesting POST request in packet 16102. In this packet, the attacker is sending a POST request to /upload.php. This could mean that the attacker is attempting to upload something with the POST request. I followed the HTTP stream for this packet to investiage further.

![POSTUploadphp Stream](/images/POSTUploadphp.png "POSTUploadphp Stream")  
*Fig 13: This screenshot shows the HTTP Stream for the POST request to /upload.php*

I was able to find out from the HTTP Stream that the attacker attempted to upload a file called dbfunctions.php in the POST request. Looking further down through the stream, I could see that the server responded with a "HTTP 200 OK" message with a response that said "The file dbfunctions.php has been uploaded" which means that the file was successfully uploaded to the server.

### Step 10:
After this, I started looking for additonal packets further down that showed any interaction with dbfunctions.php. I found a few HTTP GET requests to /uploads/dbfunctions.php in packets 16121, 16134, and 16144 which seem to be attempting to send commands to dbfunctions.php script. I found another GET request in packet 16201 that had some encoded data in the request, so I followed the HTTP stream to copy the encoded data and paste it in the decoder tool that I have previously been using.

![Packet 16201 Decoded](/images/URLDecode16201.png "Packet 16201 Decoded")
*Fig 14: This screenshot the decoded output of the GET request to /uploads/dbfunctions.php*

From the decoded output, I can tell that the attacker is using python and importing some libraries that will allow them to craft and send commands in the GET request. They initiate a socket connection that will remotely connect back to the attacker's IP 10.251.96.4 on port 4422. 

The next part of the decoded output indicates that standard input, output, and error will all be duplicated and redirected to the socket. I can tell based on the descriptors used: 0 is Standard Input, 1 is Standard Output, and 2 is Standard Error. This means that any commands, outputs or errors will appear on the attacker's IP of 10.251.96.4 rather than the server since a socket connection was initiated previously.

Finally, the last part of the output shows that a call was made to open an interactive shell on the target machine. Because of the socket connection that was made, this shell will only appear on the attacker's machine, making it more sophisticated.

### Step 11:
The next thing I checked was to see if I could find the TCP 3 way handshake process that should have been generated as a result of the socket connection that was initiated. I was able to find it only a few packets later, in packets 16203-16205. I see 10.251.96.5 send a SYN message to 10.251.96.4, 10.251.96.4 replies with a SYN ACK message, and 10.251.96.5 completes the handshake by sending an ACK reply to 10.251.96.4.

I followed the TCP stream of packet 16203 to inspect the data further.

![Reverse Shell TCP Stream](/images/ReverseShellTCPStream.png "Reverse Shell TCP Stream")
*Fig 15: This screenshot shows the TCP stream of the TCP 3 way handshake of the socket connection.*

Going through the output, it was clear that a reverse shell had been successfully created by the attacker and they had command line access to the server. Based on the output, the attacker mainly browsed through different directories and attempted to remove the dbfuncs.php, but got an "Operation not permitted" error.

I was also able to see that the user name on the server was www-data and the hostname of the server was bob-appserver.

This wraps up my analysis of this PCAP file, which concluded with the attacker successfully setting up a reverse shell to the target server.



