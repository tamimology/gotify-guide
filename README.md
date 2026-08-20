# All-in-One Stop Configuration Guide for Gotify
This guide shows every possible configuration for using Gotify locally


#### Gotify is a great notification platform that I migrated to after being a Pushover user for more than 3 years. The reason was that initially, Pushover used to provide 10k free notifications per application, but starting May 2026, they made it per user instead, which caused me out of notification for 1 month, and that is when the 10k counter resets, and I decided to use Gotify a few days before the reset day. I did explore Ntfy, but did not like it much (personal opinion)

# Table of Contents

1. <a href="#setting-up-gotify">Setting Up Gotify</a>
2. <a href="#testing-gotify">Testing Gotify</a>
3. <a href="#homeassistant-implementation">HomeAssistant Implementation</a>
4. <a href="#apprise-api-implementation">Apprise-API Implementation</a>
5. <a href="#netalertx-implementation">NetAlertX Implementation</a>
6. <a href="#watchtower-implementation">Watchtower Implementation</a>
7. <a href="#traccar-implementation">Traccar Implementation</a>
8. <a href="#synology-dsm-implementation">Synology DSM Implementation</a>
9. <a href="#synology-download-station-implementation">Synology Download Station Implementation</a>
10. <a href="#opnsense-monit-implementation">OPNSense Monit Implementation</a>




## Setting Up Gotify

Gotify is set up easily via Docker Compose as follows:

```yaml
####################################################
#                                                  #
#                 -------Gotify-------             #
#                                                  #
####################################################
#
  gotify:
    container_name: gotify
    hostname: gotify
    restart: always
    environment:
      - TZ=$TZ
      - GOTIFY_PLUGIN_DIR=/app/plugins
      - GOTIFY_DEFAULTUSER_NAME=admin
      - GOTIFY_DEFAULTUSER_PASS=$GOTIFY_USER_PASS
      # - GOTIFY_DATABASE_DIALECT=postgres
      # - GOTIFY_DATABASE_CONNECTION=host=gotify-postgres port=5432 user=gotify dbname=gotifydb password=$DB_PASSWORD sslmode=disable
      # - GOTIFY_DATABASE_DIALECT=postgres # sqlite3, mysql, postgres
      # - GOTIFY_DATABASE_CONNECTION=postgres=host=localhost port=5432 user=gotify dbname=gotifydb password=secret sslmode=disable
    volumes:
      - $PERSIST/gotify:/app/data
    networks:
      my_bridge:
    ports:
      - 9999:80
      - 1025:1025
    labels: 
      monocker.enable: true
    healthcheck:
      test: timeout 10s bash -c ':> /dev/tcp/127.0.0.1/80' || exit 1
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 90s
    image: 'gotify/server:latest'
```

If you are not familiar with Docker Compose, you may use the following Docker CLI command to run the container

`docker run --net my_bridge --name gotify -h gotify --restart always -e TZ=Australia/Melbourne -e GOTIFY_PLUGIN_DIR=/app/plugins -e GOTIFY_DEFAULTUSER_NAME=admin -e GOTIFY_DEFAULTUSER_PASS=admin -v /myvolume/docker/gotify:/app/data -p 9999:80 -p 1025:1025 -l monocker.enable=true --health-cmd "timeout 10s bash -c ':> /dev/tcp/127.0.0.1/80' || exit 1" --health-interval 10s --health-retries 3 --health-start-period 90s --health-timeout 5s gotify/server:latest`


Once it is up and running, make sure you can access it via the web by the host IP address followed by the 9999 port, i.e. 192.168.1.20:9999



## Testing Gotify

Once you have made sure that Gotify server is up and running with no issues, you need to create a new Application and take note of the token generated for that App.

#### If the generated token was not copied and saved in a safe place, then you will lose it forever and will need to re-generate a new token for this specific app.


![application-creation](/screenshots/application-creation.png)


Now, to test Gotify, send a manual notification via the following command; just replace *<apptoken>* with the token generated above

`curl "http://192.168.1.20:9999/message" -H "X-Gotify-Key: <apptoken>" -F "title=I Love Gotify" -F "message=This is a test Message" -F "priority=5"`

You should see a notification coming to the web interface. If not, double-check you captured the correct token and inserted the correct Gotify server URL and port above.



## HomeAssistant Implementation

If you are running Home Assistant and want to use Gotify as your notification platform instead of the default Companion App, then follow those steps.

There is no official Gotify integration yet; however, there are 2 methods to implement Gotify in Home Assistant.

  - ### Method 1: Using HACS integration (has not been maintained for quite some time now) 

In HACS, add this repo as *Custom Repositories* _https://github.com/1RandomDev/homeassistant-gotify_ and install the integration, then restart Home Assistant.

Once loaded, add the following in the _configuration.yaml_

```yaml
notify:
  - name: "gotify"
    platform: gotify
    url: !secret gotify_server_url # including the port
    token: !secret gotify_app_key
    verify_ssl: false #true # optional, default true
```

Verify the configuration and restart Home Assistant.

Once loaded, you may check whether it is working or not from *Settings* --> *Tools* --> *Actions*, choose _Send a notification with gotify_, and fill in the following:

- _Message_ as your message context, i.e. Testing Gotify from Home Assistant
- _Title_ as your message title, i.e. Gotify via HA
- _Target_ as your message priority, i.e. 7

Click on _Perform Action_, and you should see a notification coming to the web interface. If not, double-check that you captured the correct token and inserted the correct Gotify server URL and port above 


  - ### Method 2: Using REST API (no need to maintain as this is an independent legacy Home Assistant integration)
  
Simply add the following in the _configuration.yaml_

```yaml
notify:
  - name: gotify
    platform: rest
    resource: !secret gotify_server_url # including the port
    method: POST
    headers: 
      X-Gotify-Key: !secret gotify_app_key
    message_param_name: message
    title_param_name: title 
    target_param_name: priority
```

Verify the configuration and reload the REST integration from *Settings* --> *Tools* --> *YAML*, and scroll down till you see *REST entities and notify services* and click it

Once reloaded, you may check if it is working or not from *Settings* --> *Tools* --> *Actions*, and choose _Send a notification with gotify_, and fill in the following:

- _Message_ as your message context, i.e. Testing Gotify from Home Assistant
- _Title_ as your message title, i.e. Gotify via HA
- _Target_ as your message priority, i.e. 7

Click on _Perform Action_, and you should see a notification coming to the web interface. If not, double-check that you captured the correct token and inserted the correct Gotify server URL and port above 


I can tell that the 1st method is just an implementation of the 2nd method in a simpler way only. So both are the same, but I prefer the second as this is more reliable and there is no need to worry if HACS repo is maintained or not.


You may add as many as you want in the _notify_ section with different names and tokens to have different notification apps and categories, i.e. general notifications, 2fa code, integration-specific notification...etc



## Apprise-API Implementation

This integration will be used for several integrations, so it is a good idea to have it ready beforehand.

```yaml
####################################################
#                                                  #
#              -------AppRise-------               #
#                                                  #
####################################################
#
  apprise:
    container_name: apprise-api
    restart: always
    hostname: apprise
    environment:
      - PUID=$PUID
      - PGID=$PGID
      - TZ=$TZ
    networks:
      my_bridge:
    ports:
      - 8001:8000
    volumes:
      - $PERSIST/apprise:/config
    image: 'lscr.io/linuxserver/apprise-api:latest'
```

If you are not familiar with Docker Compose, you may use the following Docker CLI command to run the container

`docker run --net my_bridge --name apprise-api --restart always -h apprise -e PUID=1000 -e PGID=100 -e TZ=Australia/Melbourne -p 8001:8000 -v /myvolume/docker/apprise:/config lscr.io/linuxserver/apprise-api:latest`

Once loaded, navigate to `http://192.168.1.20:8001/cfg/gotify`, then click on the *CONFIGURATION* tab and insert the following
`gotify://{gotify_server_url}/{gotify_app_token}`
Just replace

- *{gotify_server_url}* with the Gotify server URL and port, i.e. _192.168.1.20:9999_
- *{gotify_token}* with the app token generated earlier

To test it, click on the *NOTIFICATIONS* tab, write anything in the _Body_ section and click on _SEND NOTIFICATION_, and you should see a notification coming to the web interface. If not, double-check that you captured the correct token and inserted the correct Gotify server domain URL above



## NetAlertX Implementation
 
NetAlertX uses the Apprise-API (check the previous step if you do not have it ready). You just need to enable the integration in NetAlertX, load it and then configure it.

To do so, just follow the steps below. Once done, click on _Save_ at the bottom of the page.


![netalertx](/screenshots/netalertx.png)


Now, navigate to *Settings* --> *Publishers* and find *Apprise publisher_ and expand it.

- _When to run_ is set to _on_notification_ to enable it.
- _Apprise host URL_ insert the created template for Gotify in Apprise, i.e. _http://192.168.1.20:8001/notify_
- _Apprise notification URL_ insert _gotify://192.168.1.20:9999/<apptoken>_



## Watchtower Implementation

Watchtower also uses the built-in _shoutrrr_ integration. To set it up, you need to have Gotify server behind a domain name, i.e. _https://gotify.mydomain.com_. Next, you need to simply add an environment variable defining this service as follows:

If using Docker Compose
```yaml
    environment:
      WATCHTOWER_NOTIFICATION_URL: "gotify://gotify.mydomain.com/$GOTIFY_DOCKER_UPDATES_TOKEN"
```

If using the Docker CLI, amend the following to your Watchtower docker run command; just make sure to replace <apptoken> with the relevant token generated

`--notification-url=gotify://gotify.mydomain.com/<apptoken>`


For more details on how to set up Watchtower, check my full compose from <a href=https://github.com/tamimology/docker-containers#watchtower>this repo</a>



## Traccar Implementation

If you have Traccar installed, then follow those steps to integrate Gotify with it. Otherwise, check <a href=https://github.com/tamimology/docker-containers#traccar>here</a> to see how to install it.

Traccar integration is a bit tricky; it is done via the SMS integration.

First, you need to add a dummy mobile number in the settings by opening Traccar and navigating to *Settings* --> *Account* and expand *Preferences* and fill as follows

![traccar](/screenshots/traccar.png)

Once it is done, navigate to the Docker container's data persist folder and open *traccar.xml* file. Navigate to the end and add the following line:

Make sure you have Gotify behind a domain name, i.e. _https://gotify.mydomain.com/_. Also replace <apptoken> with the relevant generated token earlier

```xml
    <entry key='notificator.types'>web,mail,sms</entry>
    <entry key='sms.http.template'>
        {
                "message": "{message}",
                "priority":  5
        }
    </entry>
    <entry key='sms.http.url'>https://gotify.mydomain.com/message?token=<apptoken></entry>

```

To test it, navigate to *Settings* --> *Notifications* and add a new notification by clicking on the *+* in the lower right corner. Choose the *Type* as needed, then choose _SMS_ as the *Channel*. Click on _TEST CHANNELS_, and you should see a notification coming to the web interface. If not, double-check that you captured the correct token and inserted the correct Gotify server domain URL above


![testing-traccar](/screenshots/testing-traccar.png)



## Synology DSM Implementation

Synology DSM 6.x is a bit tricky to implement as well; however, if you are using DSM 7.0 and above, then it is straightforward.

 ### - Synology DSM 7.0 and above

 This can be done in either the built-in Webhook or HTTP POST method. Refer <a href=https://mariushosting.com/how-to-install-gotify-on-your-synology-nas/>Marius Hosting Guide, step 26 onwards</a> to see how

 
 ### - Synology DSM 6.x

DSM 6.x does not have webhooks or HTTP POST built in; rather, it uses emails via SMTP to send notifications, and since Gotify is a simple post application, this is what makes it a challenge.

To solve this, you need to install a converter that captures all SMTP requests and converts them into Gotify. This is done via SMTP-to-Gotify docker conatianer, Thanks for <a href=https://github.com/lreading/smtp-to-gotify-docker>Leo</a> for his amazing repo

To start with, install the SMTP-to-Gotify Docker container via Docker Compose

```yaml
####################################################
#                                                  #
#      -------Synology-SMTP-To-Gotify-------       #
#                                                  #
####################################################
#
  synology-smtp-to-gotify:
    container_name: synology-smtp-to-gotify
    hostname: synology-smtp-to-gotify
    restart: always
    environment:
    - GOTIFY_URL=https://gotify.mydomain.com
    - GOTIFY_TOKEN=$GOTIFY_SYNOLOGY_TOKEN
    - API_KEY=$APP_KEY
    network_mode: "host"
    ports:
      - 2525:2525
    depends_on:
      gotify:
        condition: service_healthy
    image: 'imoshtokill/smtp-to-gotify-docker:latest'
```

Now, open Synology DSM, then navigate to *Control Panel* --> *Notification* --> *Email* and insert the following:

- _Enable email notifications_: Tick
- _Recipient's email address_: Add whatever email you want; no effect
- _Subject prefix_: Notification subject, i.e. My Sonolgy Server
- _Service prvider_: Custom SMTP server
- _SMTP server_: insert the host IP where you deplyed the SMTP-to-Gotify container above, i.e. 192.168.1.20
- _SMTP port_: insert the above container's port, i.e. 2525
- _Authentication required_: Tick
- _Username_: whatever you want, no effect
- _Password_: insert the API_KEY from the above container environment variables
- _Secure connection (SSL/TLS) is required_: Tick
- _Sender name_: whatever, no effect
- _Sender email_: whatever, no effect

Click on _Apply_, then *Send a test email*, and you should see a notification coming to the web interface. If not, double-check that you captured the correct token and inserted the correct Gotify server domain URL above



## Synology Download Station Implementation
#### This is applicable on DSM 6.x. I have no idea how to do it on DSM7.0 and above, not sure if the previous step is the only step needed or not.

Also, Synology Download Station uses email as its notification channel, but it is a different channel than the one we set up above for DSM. To configure this, open the *Personal* setting by clicking on the person icon in the top-right corner, then choose *Personal* --> *Email Account* tab --> *Add* --> *Customize*, and fill as follows:

### You may use the same SMTP-to-Gotify deployed above, or deploy a new one with a different port, i.e. 2526. I use 2 different containers as I like to have each app separated with its relevant icon.

- _SMTP server_: insert the host IP where you deplyed the SMTP-to-Gotify container above, i.e. 192.168.1.20
- _SMTP port_: insert the above container's port, i.e. 2525, or the newly separately deployed container's port, i.e. 2526
- _Authentication required_: Tick
- _Username_: whatever you want, no effect
- _Password_: insert the API_KEY from the above container environment variables
- _Secure connection (SSL/TLS) is required_: Tick
- _Sender name_: whatever, no effect
- _Sender email_: whatever, no effect

Click on _Apply_, do not think of testing the configuration, as it will not work and will drive you crazy. From the list, click the newly created email, and _Set as default_


Now, open *Download Station*, click the settings gear in the lower-left corner, and navigate to *Notification*.

- _Enable email notifications_: Tick
- _From_: Choose the newly created email account above, usually, it ends with *(Custom)*
- _Recipients_: Add whatever email you want; no effect

Click on _Ok_,  then _Send a test message_, and you should see a notification coming to the web interface. If not, double-check that you captured the correct token and inserted the correct Gotify server domain URL above. If you did not click _Ok_ before testing, it will not work



## OPNSense Monit Implementation

I use OPNsense as my firewall, and I like to use *Monit* as the notification integration. However, this also uses email channels, which makes it challenging. I know what we have done in Synology will not work, though, and that is why it is challenging.

To resolve it, we still need to use an SMTP-to-Gotify converter, but not via Docker. The good thing about Gotify is that it supports community-contributed plugins.

First, make sure that you exposed port 1025 when deploying the Gotify container; in my guide, I have already included this.

Second, install the SMTP-to-Gotify plugin from <a href=https://github.com/Bladeage/gotify-smtp> here </a>. Make sure to choose the correct hardware architecture build. Copy the downloaded _.so_ file into _/myvolume/docker/gotify/plugins_ folder, and restart the Gotify container.

Once loaded, open Gotify web UI, navigate to the *Plugins* tab and enable the SMTP plugin


![gotify-plugins](/screenshots/gotify-plugins.png)

Make sure you have Monit installed on your OPNsense instance before proceeding further.

Now, open OPNSense web UI, and navigate to *Services* --> *Monit* --> *Settings*, and fill as follows:

- _Enable Monit_: Tick
- _Mail Server Address_: your Gotify server URL, without the port, i.e. _192.168.1.20_
- _Mail Server Port_: 1025, cannot be changed unless you expose it (left side) differently in your Gotify Docker deployment
- _Mail Server Username_: your Gotify server username, i.e. _admin_
- _Mail Server Password_: your Gotify server password, i.e. _admin_
- _Mail Server SSL Connection_: Un-Tick

Click on _Apply_, and you should see an automatic notification coming to the web interface. If not, double-check that you captured the correct token and inserted the correct Gotify server domain URL above.







## License
This document guide is licensed under the CC0 1.0 Universal license. The terms of the license are detailed in [LICENSE](/LICENSE)
