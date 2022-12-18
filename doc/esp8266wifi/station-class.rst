:orphan:

Station Class
-------------

The number of features provided by ESP8266 in the station mode is far more extensive than covered in original `Arduino WiFi library <https://www.arduino.cc/en/Reference/WiFi>`__. Therefore, instead of supplementing original documentation, we have decided to write a new one from scratch.

Description of station class has been broken down into four parts. First discusses methods to connect to an access point (AP). Second provides methods to manage the connection like e.g. ``reconnect`` or ``isConnected``. Third covers properties to obtain information about the connection like MAC or IP address. Finally the fourth section provides alternate methods to connect like e.g. Wi-Fi Protected Setup (WPS).

Table of Contents
-----------------

-  `Connect to Wifi <#start-here>`__

   -  `begin <#begin>`__
   -  `config <#config>`__
   -  `SDK Connect <#sdk-connect>`__

-  `Manage Connection <#manage-connection>`__

   -  `reconnect <#reconnect>`__
   -  `disconnect <#disconnect>`__
   -  `isConnected <#isconnected>`__
   -  `setAutoConnect <#setautoconnect>`__
   -  `getAutoConnect <#getautoconnect>`__
   -  `setAutoReconnect <#setautoreconnect>`__
   -  `waitForConnectResult <#waitforconnectresult>`__

-  `Configuration <#configuration>`__

   -  `macAddress <#macAddress>`__
   -  `localIP <#localip>`__
   -  `subnetMask <#subnetmask>`__
   -  `gatewayIP <#gatewayip>`__
   -  `dnsIP <#dnsip>`__
   -  `hostname <#hostname>`__
   -  `status <#status>`__
   -  `SSID <#ssid>`__
   -  `psk <#psk>`__
   -  `BSSID <#bssid>`__
   -  `RSSI <#rssi>`__

-  `Connect Different <#connect-different>`__

   -  `WPS <#wps>`__
   -  `Smart Config <#smart-config>`__

Points below provide description and code snippets how to use particular methods.

For more code samples please refer to separate section with `examples <station-examples.rst>`__ dedicated specifically to the Station Class.

Start to Connect Wifi
~~~~~~~~~~~~~~~~~~~~~

Add a line to set the device into a known wifi mode, start with ``WiFi.mode(WIFI_STA)``. SoftAP mode can be explored later. Typical parameters passed to ``begin`` include SSID and passphrase, so module can connect to specific Access Point.

.. code:: cpp

    WiFi.begin(ssid, passphrase)

begin
^^^^^

There are several ``begin`` `function overloads <https://en.wikipedia.org/wiki/Function_overloading>`__ (versions). One was presented just above:
``WiFi.begin(ssid, passphrase)``. Overloads provide flexibility in number or type of accepted parameters. They are also the easiest way to update the 'wifi config' saved in flash. -- see `SDK Connect <#sdk-connect>`__ below.

The simplest overload of ``begin`` is as follows:

.. code:: cpp

    WiFi.begin()

Calling it will enable station mode and connect to the last access point saved in flash memory.

Notes:

- It is possible that calling ``begin`` will result in the module being in STA + softAP mode if the module was previously placed into AP mode. 
- If you notice strange behavior with DNS or other network functionality, check which mode your module is in (see ``WiFi.getMode()`` in the `Generic Class Documentation <generic-class.rst#mode>`__).

Below is the syntax of another overload of ``begin`` with the all possible parameters:

.. code:: cpp

    WiFi.begin(ssid, passphrase, channel, bssid, connect)

Meaning of parameters is as follows:

- ``ssid`` - a character string containing the SSID of Access Point we would like to connect to, may have up to 32 characters
- ``passphrase`` to the access point, strlen max 63, or 64 if a hexadecimal string, while it can be minimum 8, circa core 3, 18 would be appropriate.
- ``channel`` of AP, if we like to operate using specific channel, some may be faster than others, if this parameter is omitted, time is used to negotiate.
- ``bssid`` - mac address of AP, this parameter is also optional, but will make connections quicker, as some net scanning may be skipped.
- ``connect`` - a ``boolean`` parameter that if set to ``false``, will instruct module just to save the other parameters without actually establishing connection to the access point

config
^^^^^^

Disable `DHCP <https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol>`__ client (Dynamic Host Configuration Protocol) and set the IP configuration of station interface to user defined arbitrary values. The interface will be a static IP configuration instead of values provided by DHCP.

.. code:: cpp

    WiFi.config(local_ip, gateway, subnet, dns1, dns2)

Function will return ``true`` if configuration change is applied successfully. If configuration can not be applied, because e.g. module is not in station or station + soft access point mode, then ``false`` will be returned.

The following IP configuration may be provided:

-  ``local_ip`` - enter here IP address you would like to assign the ESP
   station's interface
-  ``gateway`` - should contain IP address of gateway (a router) to
   access external networks
-  ``subnet`` - this is a mask that defines the range of IP addresses of
   the local network
-  ``dns1``, ``dns2`` - optional parameters that define IP addresses of
   Domain Name Servers (DNS) that maintain a directory of domain names
   (like e.g. *www.google.co.uk*) and translate them for us to IP
   addresses

*Example code:*

.. code:: cpp

    #include <ESP8266WiFi.h>

    const char* ssid = "********";
    const char* passphrase = "****************";

    IPAddress staticIP(192,168,1,22);
    IPAddress gateway(192,168,1,9);
    IPAddress subnet(255,255,255,0);

    void setup(void)
    {
      Serial.begin(115200);
      Serial.println();

      Serial.printf("Connecting to %s\n", ssid);
      WiFi.config(staticIP, gateway, subnet);
      WiFi.begin(ssid, passphrase);
      while (WiFi.status() != WL_CONNECTED)
      {
        delay(500);
        Serial.print(".");
      }
      Serial.println();
      Serial.print("Connected, IP address: ");
      Serial.println(WiFi.localIP());
    }

    void loop() {}

*Example output:*

::

    Connecting to sensor-net
    .
    Connected, IP address: 192.168.1.22

Please note that station with static IP configuration usually connects to the network faster. In the above example it took about 500ms (one dot `.` displayed). This is because obtaining of IP configuration by DHCP client takes time and in this case this step is skipped. If you pass all three parameter as 0.0.0.0 (local_ip, gateway and subnet), it will re enable DHCP. You need to re-connect the device to get new IPs.


SDK Connect
^^^^^^^^^^^

SDK auto connect can be twice as fast as begin, partly because it runs before user code. How fast? Expect 1st connection around the 220 ms mark, while reconnects take about 160 ms, on a not very busy wlan with a signal strength about -60dB. The SDK connect method is valuable to projects that demand the quickest wifi ready. For example, if battery powered, the esp8266 can turn off the radio about a 1/4 second sooner than with begin.

SDK connect completely relies on the correct wifi settings saved in flash. If the settings need updating, we can call begin one time. We don't even have to connect (5th param false as in the example code below). The more args you pass to begin, the quicker the connections will be.

WiFi.config can also make SDK connect a little quicker, but it really helps begin much more. Try it.

*Example code:*

.. code:: cpp

   #define MS Serial.print(millis());  Serial.write(' ');

   #include <ESP8266WiFi.h>

   const char* ssid        = "********";                           // max strlen 32
   const char* passkey     = "****************";                   // max strlen 63, or 64 if hexadecimal string
   int8_t      channel     = 1;                                    // choose the fastest/best on local wlan
   uint8_t     bssid[6]    = {0xA4, 0xB1, 0xE9, 0xCD, 0x6B, 0x29}; // can use wifiscan example, or AP's web mgmt site, to get bssid

   IPAddress staIP         = {192,168,1,69};
   IPAddress gateway       = {192,168,1,254};
   IPAddress subnet        = {255,255,255,0};

   void setup()
   {
       Serial.begin(115200);
       enableWiFiAtBootTime();  // prevents shutdown of sdk connect, obviates calling persistent(true)
       //Serial.setDebugOutput(false);  // default true since core 3.0
       if (! WiFi.config(staIP, gateway, subnet)) {
           Serial.println(F("WiFi.config failed; DHCP will add ~2 sec to connect time; check the static IPs."));
       }

       // Do we need to call begin to write new wifi settings in flash?
       //  Only if sketch & flash settings are different (changed), else just wait for sdk to connect
       struct station_config wl_args;
       wifi_station_get_config (&wl_args);
       if (strcmp(reinterpret_cast<const char*>(wl_args.ssid), ssid) != 0 ||
           strcmp(reinterpret_cast<const char*>(wl_args.password), passkey) != 0) {          // need to erase/rewrite station_config
           if (WiFi.getMode() != 1) WiFi.mode(WIFI_STA);
           WiFi.persistent(true);          // needed persist(true) or enableWiFiAtBootTime(), or settings not saved to flash
           wl_status_t ret = WiFi.begin(ssid, passkey, channel, bssid, false);  // do not connect, but write flash if different
           MS Serial.printf(PSTR("Wifi args updated in flash, ssid='%s' passkey='%s' channel=%d bssid=" MACSTR),
                                                                   ssid, passkey, channel, MAC2STR(bssid));
           ESP.restart();  // Restarting to test newly updated station_config"));
       }
   }

   void loop()
   {
       static bool waitWifi = true;
       if (WiFi.status() == WL_CONNECTED && waitWifi) {  // async wait, do something in the ms you wait for wifi
           MS Serial.println("WL_CONNECTED");
           // cycle wifi mode thru off back to sta, adds about 190 ms here to slow down this demo
           // WiFi.mode(WIFI_OFF);  WiFi.mode(WIFI_STA);  // comment to run full speed, OFF disconnects but does not erase flash wifi settings
           waitWifi = WiFi.reconnect();
           MS Serial.println("Attempting to reconnect wifi...");
       }
   }

*Example output:*

::

   216 WL_CONNECTED
   223 Attempting to reconnect wifi...
   377 WL_CONNECTED



Manage Connection
~~~~~~~~~~~~~~~~~

reconnect
^^^^^^^^^

Reconnect the station. This is done by disconnecting from the access point an then initiating connection back to the same AP. 
By default, ESP will attempt to reconnect to Wi-Fi network whenever it is disconnected. There is no need to handle this by separate code. A good way to simulate disconnection would be to reset the access point. ESP will report disconnection, and then try to reconnect automatically.


.. code:: cpp

    bool ret = WiFi.reconnect();

Notes: 1. Station should be already connected to an access point. If this is not the case, then function will return ``false`` not performing any action. 2. If ``true`` is returned it means that connection sequence has been successfully started. User should still check for connection status, waiting until ``WL_CONNECTED`` is reported:

.. code:: cpp

    if (WiFi.reconnect()) {
       while (WiFi.status() != WL_CONNECTED)
       {
         delay(500);
         Serial.print(".");
       }
    }

disconnect
^^^^^^^^^^

Sets currently configured SSID and passphrase to ``null`` values and disconnects the station from an access point.

.. code:: cpp

    WiFi.disconnect(wifioff)

The ``wifioff`` is an optional ``boolean`` parameter. If set to ``true``, then the station mode will be turned off.

isConnected
^^^^^^^^^^^

Returns ``true`` if Station is connected to an access point or ``false`` if not.

.. code:: cpp

    WiFi.isConnected()

setAutoConnect
^^^^^^^^^^^^^^

Configure module to automatically connect on power on to the last used access point.

.. code:: cpp

    WiFi.setAutoConnect(autoConnect)

The ``autoConnect`` is an optional parameter. If set to ``false`` then auto connection functionality up will be disabled. If omitted or set to ``true``, then auto connection will be enabled.

getAutoConnect
^^^^^^^^^^^^^^

This is "companion" function to ``setAutoConnect()``. It returns ``true`` if module is configured to automatically connect to last used access point on power on.

.. code:: cpp

    WiFi.getAutoConnect()

If auto connection functionality is disabled, then function returns ``false``.

setAutoReconnect
^^^^^^^^^^^^^^^^

Set whether module will attempt to reconnect to an access point in case it is disconnected.

.. code:: cpp

    WiFi.setAutoReconnect(autoReconnect)

If parameter ``autoReconnect`` is set to ``true``, then module will try to reestablish lost connection to the AP. If set to ``false`` then module will stay disconnected.

Note: running ``setAutoReconnect(true)`` when module is already disconnected will not make it reconnect to the access point. Instead ``reconnect()`` should be used.

waitForConnectResult
^^^^^^^^^^^^^^^^^^^^

Wait until module connects to the access point. This function is intended for module configured in station or station + soft access point mode.

.. code:: cpp

    WiFi.waitForConnectResult()

Function returns one of the following connection statuses: 

- ``WL_CONNECTED`` after successful connection is established 
- ``WL_NO_SSID_AVAIL`` in case configured SSID cannot be reached
- ``WL_CONNECT_FAILED`` if connection failed 
- ``WL_CONNECT_WRONG_PASSWORD`` if passphrase is > 64 chars
- ``WL_IDLE_STATUS`` when Wi-Fi is in process of changing between statuses 
- ``WL_DISCONNECTED`` if module is not configured in station mode
- ``-1`` on timeout

Configuration
~~~~~~~~~~~~~

macAddress
^^^^^^^^^^

Get the MAC address of the ESP station's interface.

.. code:: cpp

    WiFi.macAddress(mac)

Function should be provided with ``mac`` that is a pointer to memory location (an ``uint8_t`` array the size of 6 elements) to save the mac address. The same pointer value is returned by the function itself.

*Example code:*

.. code:: cpp

    if (WiFi.status() == WL_CONNECTED)
    {
      uint8_t macAddr[6];
      WiFi.macAddress(macAddr);
      Serial.printf("Connected, mac address: %02x:%02x:%02x:%02x:%02x:%02x\n", macAddr[0], macAddr[1], macAddr[2], macAddr[3], macAddr[4], macAddr[5]);
    }

*Example output:*

::

    Mac address: 5C:CF:7F:08:11:17

If you do not feel comfortable with pointers, then there is optional version of this function available. Instead of the pointer, it returns a formatted ``String`` that contains the same mac address.

.. code:: cpp

    WiFi.macAddress()

*Example code:*

.. code:: cpp

    if (WiFi.status() == WL_CONNECTED)
    {
      Serial.printf("Connected, mac address: %s\n", WiFi.macAddress().c_str());
    }

localIP
^^^^^^^

Function used to obtain IP address of ESP station's interface.

.. code:: cpp

    WiFi.localIP()

The type of returned value is `IPAddress <https://github.com/esp8266/Arduino/blob/master/cores/esp8266/IPAddress.h>`__. There is a couple of methods available to display this type of data. They are presented in examples below that cover description of ``subnetMask``, ``gatewayIP`` and ``dnsIP`` that return the IPAdress as well.

*Example code:*

.. code:: cpp

    if (WiFi.status() == WL_CONNECTED)
    {
      Serial.print("Connected, IP address: ");
      Serial.println(WiFi.localIP());
    }

*Example output:*

::

    Connected, IP address: 192.168.1.10

subnetMask
^^^^^^^^^^

Get the subnet mask of the station's interface.

.. code:: cpp

    WiFi.subnetMask()

Module should be connected to the access point to obtain the subnet mask.

*Example code:*

.. code:: cpp

    Serial.print("Subnet mask: ");
    Serial.println(WiFi.subnetMask());

*Example output:*

::

    Subnet mask: 255.255.255.0

gatewayIP
^^^^^^^^^

Get the IP address of the gateway.

.. code:: cpp

    WiFi.gatewayIP()

*Example code:*

.. code:: cpp

    Serial.printf("Gataway IP: %s\n", WiFi.gatewayIP().toString().c_str());

*Example output:*

::

    Gataway IP: 192.168.1.9

dnsIP
^^^^^

Get the IP addresses of Domain Name Servers (DNS).

.. code:: cpp

    WiFi.dnsIP(dns_no)

With the input parameter ``dns_no`` we can specify which Domain Name Server's IP we need. This parameter is zero based and allowed values are none, 0 or 1. If no parameter is provided, then IP of DNS #1 is returned.

*Example code:*

.. code:: cpp

    Serial.print("DNS #1, #2 IP: ");
    WiFi.dnsIP().printTo(Serial);
    Serial.print(", ");
    WiFi.dnsIP(1).printTo(Serial);
    Serial.println();

*Example output:*

::

    DNS #1, #2 IP: 62.179.1.60, 62.179.1.61

hostname
^^^^^^^^

Get the DHCP hostname assigned to ESP station.

.. code:: cpp

    WiFi.hostname()

Function returns ``String`` type. Default hostname is in format ``ESP_24xMAC`` where 24xMAC are the last 24 bits of module's MAC address.

The hostname may be changed using the following function:

.. code:: cpp

    WiFi.hostname(aHostname)

Input parameter ``aHostname`` may be a type of ``char*``, ``const char*`` or ``String``. Maximum length of assigned hostname is 32 characters. Function returns either ``true`` or ``false`` depending on result. For instance, if the limit of 32 characters is exceeded, function will return ``false`` without assigning the new hostname.

*Example code:*

.. code:: cpp

    Serial.printf("Default hostname: %s\n", WiFi.hostname().c_str());
    WiFi.hostname("Station_Tester_02");
    Serial.printf("New hostname: %s\n", WiFi.hostname().c_str());

*Example output:*

::

    Default hostname: ESP_081117
    New hostname: Station_Tester_02

status
^^^^^^

Returns the status of the wifi connection.

.. code:: cpp

    WiFi.status()

One of the following values of type of ``wl_status_t`` as defined in `wl\_definitions.h <https://github.com/esp8266/Arduino/blob/master/cores/esp8266/wl_definitions.h>`__

- ``WL_IDLE_STATUS`` 0, when status is in process of changing
- ``WL_NO_SSID_AVAIL`` 1, configured SSID cannot be reached
- ``WL_SCAN_COMPLETED`` 2,
- ``WL_CONNECTED`` 3, wifi connected
- ``WL_CONNECT_FAILED`` 4, 
- ``WL_CONNECTION_LOST`` 5,
- ``WL_WRONG_PASSWORD`` 6, passphrase is too long
- ``WL_DISCONNECTED`` 7, wifi is on, but not connected to an access point

``wl_status_t`` is also the return type of other WiFi methods.
.. code:: cpp

    wl_status_t status = WiFi.begin();
    wl_status_t status = WiFi.waitForConnectResult();


*Example code:*

.. code:: cpp

    #include <ESP8266WiFi.h>
    
    const char *ssid = "sensor-net";
    const char *passphrase = "Planetary_Unique pa55phrase";

    void setup(void)
    {
      Serial.begin(115200);
      Serial.printf("Connection status: %d\n", WiFi.status());
      Serial.printf("Connecting to %s\n", ssid);
      wl_status_t status = WiFi.begin(ssid, passphrase);
      Serial.printf("WiFi.begin returned status: %d\n", status);
      while (WiFi.status() != WL_CONNECTED)
      {
        delay(500);
        Serial.print(".");
      }
      Serial.printf("\nConnection status: %d\n", WiFi.status());
      Serial.print("Connected, IP address: ");
      Serial.println(WiFi.localIP());
    }

    void loop() {}

*Example output:*

::

    Connection status: 7
    Connecting to sensor-net
    WiFi.begin returned status: 7
    ......
    Connection status: 3
    Connected, IP address: 192.168.1.10

::

    3 - WL_CONNECTED
    7 - WL_DISCONNECTED

See `status <#status>`__ for other return values.


SSID
^^^^

Return the name of Wi-Fi network, formally called `Service Set Identification (SSID) <https://www.juniper.net/techpubs/en_US/network-director1.1/topics/concept/wireless-ssid-bssid-essid.html#jd0e34>`__.

.. code:: cpp

    WiFi.SSID()

Returned value is of the ``String`` type.

*Example code:*

.. code:: cpp

    Serial.printf("SSID: %s\n", WiFi.SSID().c_str());

*Example output:*

::

    SSID: sensor-net

psk
^^^

Return current pre shared key (passphrase) associated with the Wi-Fi network.

.. code:: cpp

    WiFi.psk()

Function returns value of the ``String`` type.

BSSID
^^^^^

Return the mac address of the access point to which the ESP module was directed to connect to. This address is formally called `Basic Service Set Identification (BSSID) <https://www.juniper.net/techpubs/en_US/network-director1.1/topics/concept/wireless-ssid-bssid-essid.html#jd0e47>`__. The returned pointer is what the user configured when calling begin() with a bssid argument. It does _not_ necessarily reflect the mac address of the access point to which the ESP module's station interface is currently connected to.

.. code:: cpp

    WiFi.BSSID()

The ``BSSID()`` function returns a pointer to the memory location (an ``uint8_t`` array with the size of 6 elements) where the BSSID is saved.

Below is similar function, but returning BSSID but as a ``String`` type.

.. code:: cpp

    WiFi.BSSIDstr()

*Example code:*

.. code:: cpp

    Serial.printf("BSSID: %s\n", WiFi.BSSIDstr().c_str());

*Example output:*

::

    BSSID: 00:1A:70:DE:C1:68

RSSI
^^^^

Return the signal strength of Wi-Fi network, that is formally called `Received Signal Strength Indication (RSSI) <https://en.wikipedia.org/wiki/Received_signal_strength_indication>`__.

.. code:: cpp

    WiFi.RSSI()

Signal strength value is provided in dBm. The type of returned value is ``int32_t``.

*Example code:*

.. code:: cpp

    Serial.printf("RSSI: %d dBm\n", WiFi.RSSI());

*Example output:*

::

    RSSI: -68 dBm

Connect Different
~~~~~~~~~~~~~~~~~

`ESP8266 SDK <https://bbs.espressif.com/viewtopic.php?f=51&t=1023>`__ provides alternate methods to connect ESP station to an access point. Out of them `esp8266 / Arduino <https://github.com/esp8266/Arduino>`__ core implements `WPS <#wps>`__ and `Smart Config <#smart-config>`__ as described in more details below.

WPS
^^^

The following ``beginWPSConfig`` function allows connecting to a network using `Wi-Fi Protected Setup (WPS) <https://en.wikipedia.org/wiki/Wi-Fi_Protected_Setup>`__. Currently only `push-button configuration <https://www.wi-fi.org/knowledge-center/faq/how-does-wi-fi-protected-setup-work>`__ (``WPS_TYPE_PBC`` mode) is supported (SDK 1.5.4).

.. code:: cpp

    WiFi.beginWPSConfig()

Depending on connection result function returns either ``true`` or ``false`` (``boolean`` type).

*Example code:*

.. code:: cpp

    #include <ESP8266WiFi.h>

    void setup(void)
    {
      Serial.begin(115200);
      Serial.println();

      Serial.printf("Wi-Fi mode set to WIFI_STA %s\n", WiFi.mode(WIFI_STA) ? "" : "Failed!");
      Serial.print("Begin WPS (press WPS button on your router) ... ");
      Serial.println(WiFi.beginWPSConfig() ? "Success" : "Failed");

      while (WiFi.status() != WL_CONNECTED)
      {
        delay(500);
        Serial.print(".");
      }
      Serial.println();
      Serial.print("Connected, IP address: ");
      Serial.println(WiFi.localIP());
    }

    void loop() {}

*Example output:*

::

    Wi-Fi mode set to WIFI_STA
    Begin WPS (press WPS button on your router) ... Success
    .........
    Connected, IP address: 192.168.1.102

Smart Config
^^^^^^^^^^^^

The Smart Config connection of an ESP module an access point is done by sniffing for special packets that contain SSID and passphrase of desired AP. To do so the mobile device or computer should have functionality of broadcasting of encoded SSID and passphrase.

The following three functions are provided to implement Smart Config.

Start smart configuration mode by sniffing for special packets that contain SSID and passphrase of desired Access Point. Depending on result either ``true`` or ``false`` is returned.

.. code:: cpp

    beginSmartConfig()

Query Smart Config status, to decide when stop configuration. Function returns either ``true`` or ``false`` of ``boolean`` type.

.. code:: cpp

    smartConfigDone()

Stop smart config, free the buffer taken by ``beginSmartConfig()``. Depending on result function return either ``true`` or ``false`` of ``boolean`` type.

.. code:: cpp

    stopSmartConfig()

For additional details regarding Smart Config please refer to `ESP8266 API User Guide <https://bbs.espressif.com/viewtopic.php?f=51&t=1023>`__.
