:orphan:

Station Class
-------------

The number of features provided by ESP8266 in the station mode is far more extensive than covered in original `Arduino WiFi library <https://www.arduino.cc/en/Reference/WiFi>`__. Therefore, instead of supplementing original documentation, we have decided to write a new one from scratch. This page is based on the `source code <https://github.com/esp8266/Arduino/>`__.

Description of station class has been broken down into four parts. First discusses methods to connect to an access point (AP). Second provides methods to manage the connection like e.g. ``reconnect`` or ``isConnected``. Third covers properties to obtain information about the connection like MAC or IP address. Finally the fourth section provides alternate methods to connect like e.g. Wi-Fi Protected Setup (WPS).

Table of Contents
-----------------

-  `Connect Wifi <#connect-wifi>`__

   -  `begin <#begin>`__
   -  `config <#config>`__
   -  `SDK connect <#sdk-connect>`__

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


Connect Wifi
~~~~~~~~~~~~

Before connecting a module to a wifi access point (AP), set the module's wifi to a known `mode <generic-class.rst#mode>`__. Many modules start in WIFI_AP (SoftAP) mode, so then calling ``begin``, which adds station mode, would place the device in the dual sta + softAP mode. For now, only the station mode is needed

.. code:: cpp

    WiFi.mode(WIFI_STA);


begin
^^^^^

There are several ``begin`` `function overloads <https://en.wikipedia.org/wiki/Function_overloading>`__. Overloads provide flexibility in number or type of accepted parameters. ``WiFi.begin`` returns wifi connection `status <#status>`__.

.. code:: cpp

    WiFi.begin()
    WiFi.begin(ssid, passphrase)
    WiFi.begin(ssid, passphrase, channel, bssid)
    WiFi.begin(ssid, passphrase, channel, bssid, connect)

Meaning of optional parameters is as follows:

- ``ssid`` - a string containing the SSID of the AP to connect to, max string length is 32 characters.
- ``passphrase`` to the AP, strlen max 63, or 64 if a hexadecimal string, while the minimum is 8, in the early 2020's 18 would be better.
- ``channel`` may depend on AP settings, how busy the channels are, specify 0 (zero) if unsure and your device and the AP should negotiate a channel.
- ``bssid`` - mac address of AP, but will make connections quicker, as some wifi scanning may be skipped.
- ``connect`` - a boolean (true/false) parameter by default true, after saving the wifi settings, attempt to connect to the AP.

Calling begin with args, can result in those args being saved in the 'wifi settings' area of flash. Calling ``WiFi.begin()`` without args, causes the module to use those saved settings. See `SDK Connect <#sdk-connect>`__ for an alternative to begin.

For now, use the slower, but more flexible overload:

.. code:: cpp

    WiFi.begin(ssid, passphrase);


config
^^^^^^

Applies an IP address configuration to a module operating in station mode. Non-zero addresses specify a static configuration, which disables the Dynamic Host Configuration Protocol `(DHCP <https://wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol>`__) client. This reduces the time to connect, as DHCP client-server IP conversation is obviated. ``WiFi.config`` is also indispensable when the module needs the same IP address each time it starts.

Choose an address for the station that is in the same subnet as the gateway, but outside any DHCP pools used by the AP (router) in that subnet, or IP address conflicts may arise.

``WiFi.config`` returns bool (true or false). If the IP addresses were recorded successfully, it returns ``true``, otherwise ``false``. Misconfiguration or typos can cause failure, if for example the sta_ip and gateway are specified in different subnets. See the following example code.

``WiFi.config``, even without overloads, may be called serveral ways:

.. code:: cpp

    WiFi.config(sta_ip, gateway, subnet)
    WiFi.config(sta_ip, gateway, subnet, dns1)
    WiFi.config(sta_ip, gateway, subnet, dns1, dns2)

Meaning of parameters is as follows (first three are required):

-  ``sta_ip`` - address you would like assigned to the station's interface
-  ``gateway`` - address of a gateway (router) on the station's subnet, provides access to external hosts (beyond the local subnet)
-  ``subnet`` - a mask that defines the range of IP addresses of the local network
-  ``dns1`` - address of a Domain Name System (`DNS <https://wikipedia.org/wiki/Domain_Name_System>`__) server, required to get addresses of external or public hostnames, eg. github.com
-  ``dns2`` - optional, address of a 2nd DNS server

*Example code:*

.. code:: cpp

    IPAddress sta_ip (192,168,1,22);
    IPAddress gateway (192,186,1,9);         // <- misconfiguration 
    IPAddress subnet (255,255,255,0);
    IPAddress dns1 = gateway;                // required to get IP of external hostnames

    void setup(void)
    {
      ...

      if (! WiFi.config (sta_ip, gateway, subnet, dns1)) {
         Serial.println(F("WiFi.config failed; check the defined IPs; falling back to DHCP"));
      }

      ...
    }

*Example output:*

::

   WiFi.config failed; check the defined IPs; falling back to DHCP

``WiFi.config`` can revert an IP address configuration from static to dynamic. If the station requires a new DHCP configuration, the DHCP client can be reactivated by specifying zero addresses, and the new configuration is obtained with a call to ``WiFi.reconnect()``:

*Example code:*

.. code:: cpp

    if (! WiFi.config ({0,0,0,0},{0,0,0,0},{0,0,0,0})) {
        Serial.println(F("WiFi.config could not reenable DHCP client"));
    } else {
        Serial.println(F("Calling WiFi.reconnect()..."));
        if (! WiFi.reconnect()) {
            Serial.println(F("Failed to initiate WiFi.reconnect."));
        } else {
            Serial.println(F("Blocking while reconnecting..."));
            while (WiFi.status() != WL_CONNECTED) { yield(); }
            if (-1 == uart_get_debug()) {           // UART_NO(-1), UART0(0), UART1(1), print if setDebugOutput(false)
               Serial.printf(PSTR("Station IP: %s\n"), WiFi.localIP().toString().c_str());
               Serial.printf(PSTR("Gataway IP: %s\n"), WiFi.gatewayIP().toString().c_str());
               Serial.printf(PSTR("SubnetMask: %s\n"), WiFi.subnetMask().toString().c_str());
            }
            Serial.print(F("DNS servers: (0):"));  Serial.print(WiFi.dnsIP(0));
            Serial.print(F(", (1):"));  Serial.println(WiFi.dnsIP(1));
        }
    }


SDK Connect
^^^^^^^^^^^

The SDK can connect 2x quicker than ``begin``, partly because it runs before user code. How fast? To a not very busy AP with good signal strength (-60dB), expect modules to connect around the 220 millisecond mark, while later reconnects may take about 160 ms. The SDK connect method is valuable to projects that demand the quickest IP ready module. For example, battery powered ESPs can save mAh by turn off the radio about a 1/4 second sooner than when using ``begin``.

Call ``WiFi.config`` early in setup() to avoid the SDK starting the DHCP client, wasting time, and maybe even getting an unexpected IP address.

SDK connect relies on correct wifi settings saved in flash. If the settings need updating, call ``begin`` one time. The module does not have to connect to the AP (5th param false as in the following example code), but as noted in the `begin section <#begin>`__ above, a channel and bssid should be passed for the quickest connections.


*Example code:*

.. code:: cpp

   #define MS Serial.print(millis());  Serial.print(" ms ");

   #include <ESP8266WiFi.h>

   const char* ssid        = "********";                           // max strlen 32
   const char* passkey     = "****************";                   // max strlen 63, or 64 if hexadecimal string
   int8_t      channel     = 1;                                    // choose the fastest/best on local wlan
   uint8_t     bssid[6]    = {0xA4, 0xB1, 0xE9, 0xCD, 0x6B, 0x29}; // get with wifiscan example, or AP's web site

   IPAddress staIP         = {192,168,1,50};
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
       //  Only if sketch & flash settings are different, else just wait for sdk to connect
       struct station_config wl_args;
       wifi_station_get_config (&wl_args);
       if (strcmp(reinterpret_cast<const char*>(wl_args.ssid), ssid) != 0 ||
           strcmp(reinterpret_cast<const char*>(wl_args.password), passkey) != 0) {  // need to erase/rewrite station_config
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
       if (WiFi.status() == WL_CONNECTED && waitWifi) {  // cooperative (async) wait, check a sensor or something while waiting
           MS Serial.println("WL_CONNECTED");
           // uncomment next line to slow reconnects an additional 190 ms; less stress on eyes and serial monitor :)
           // WiFi.mode(WIFI_OFF);  WiFi.mode(WIFI_STA);  // WIFI_OFF does not erase flash settings
           waitWifi = WiFi.reconnect();
           MS Serial.println("Attempting to reconnect wifi...");
       }
   }

*Example output:*

::

   216 ms WL_CONNECTED
   223 ms Attempting to reconnect wifi...
   377 ms WL_CONNECTED



Manage Connection
~~~~~~~~~~~~~~~~~

reconnect
^^^^^^^^^

Reconnect the station. This is done by disconnecting from the access point and then initiating connection back to the same AP. 
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

Wait until module connects to the access point. This function is intended for modules in station, or station + softAP, wifi mode. ``waitForConnectResult()`` blocks code processing while waiting for a wifi connection.

.. code:: cpp

    WiFi.waitForConnectResult()

Returns wifi connection `status <#status>`__


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

- ``WL_IDLE_STATUS``       0, when status is in process of changing
- ``WL_NO_SSID_AVAIL``     1, configured SSID cannot be reached
- ``WL_SCAN_COMPLETED``    2,
- ``WL_CONNECTED``         3, wifi connected
- ``WL_CONNECT_FAILED``    4, 
- ``WL_CONNECTION_LOST``   5,
- ``WL_WRONG_PASSWORD``    6, passphrase is too long
- ``WL_DISCONNECTED``      7, wifi is on, but not connected to an access point

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
