ESP32 Wi-Fi Security Lab --- Captive Portal & Rogue AP Demonstration
Security notice: This source contains functionality associated
with Evil Twin/captive-portal credential harvesting and a
deauthentication feature flag. Do not deploy it against networks or
people you do not own or have explicit written authorization to test.
This README documents the code for security analysis; it does not
provide instructions for collecting or using other people's Wi-Fi
passwords.

1. Project overview
This project is an ESP32-based wireless security demonstration. It
combines:
- A Wi-Fi access point hosted by the ESP32.
- A DNS server that directs DNS requests to the ESP32.
- An HTTP server that presents a browser page.
- Periodic scanning of nearby Wi-Fi networks.
- A web interface that displays discovered SSIDs, BSSIDs, and
  channels.
- A captive-portal-style page that imitates a router recovery or
  firmware-update prompt.
The source also contains a deauthentication state and
packet-construction code. However, the actual raw-frame transmission
calls in that section are commented out. Therefore, this version does
not actively transmit the constructed deauthentication frames
through those calls.
2. How the program is structured
                  ESP32
                    |
          Wi-Fi AP + Station mode
                    |
          +---------+----------+
          |                    |
      DNS server           HTTP server
       port 53              port 80
          |                    |
  DNS requests redirected   Browser requests
  to ESP32 AP address       handled by web routes
                               |
                     +---------+----------+
                     |                    |
                Network scan         Web interface
                     |                    |
              SSID/BSSID/channel     Select a network
                                          |
                                Captive-portal page
3. Main libraries
  Library                             Purpose
  Arduino.h                         Arduino core functions and types

  WiFi.h                            ESP32 Wi-Fi station and
access-point functions

  WebServer.h                       Hosts the local HTTP interface
DNSServer.h                       Handles DNS requests for the
captive-portal demonstration
4. Important data structures
_Network
The _Network structure stores information about one scanned wireless
network:

  Field        Meaning
  ssid       Network name
ch         Wi-Fi channel
bssid[6]   Access point MAC address

The program allocates space for up to 16 network entries:
_Network _networks[16];
_Network _selectedNetwork;
_selectedNetwork holds the network selected in the web interface.
State variables
- hotspot_active: indicates whether the alternate access-point mode
  is marked active.
- deauthing_active: tracks the deauthentication state flag.
- _correct: stores a status string created after a successful
  station connection.
- _tryPassword: stores the password submitted through the web form.
Credential-handling warning: _tryPassword is retained in memory
and included in a serial-console message when the station connection
succeeds. This is credential collection behavior. A safe educational
implementation should never collect, retain, display, or transmit real
users' Wi-Fi passwords.
5. Startup: setup()
The setup() function:
1. Starts serial communication at 115200 baud.
2. Enables Wi-Fi AP + station mode using WIFI_AP_STA.
3. Configures the ESP32 access-point network.
4. Starts an access point with a hard-coded SSID and password.
5. Starts the DNS server on port 53.
6. Registers HTTP routes.
7. Starts the web server on port 80.
The configured access-point address is 192.168.4.1, with a /24
subnet mask.

The DNS server is configured to answer requests using the ESP32
access-point address. This is the captive-portal-style redirection
behavior.
6. Network scanning: performScan()
The scanning function calls:
WiFi.scanNetworks();
It clears the previous results and stores up to 16 discovered networks.
For each result, it records:
- SSID
- BSSID
- Channel
The scan is scheduled from loop() approximately every 15 seconds.

The scan results are displayed in the web interface. Scanning nearby
networks is passive discovery, but the information should still be
handled responsibly.
7. HTTP routes and handlers
  Route / handler                     Role in this source
  / --- handleIndex()             Displays the network list and the
captive-portal-style page

  /result --- handleResult()      Checks the station connection state
and displays a result

  /admin --- handleAdmin()        Displays a similar
network-management page
Not found                           Falls back to handleIndex()
The HTML page is assembled in C++ strings. The page uses a
router-recovery/firmware-update message and contains a password input.
That design can mislead users into submitting credentials, so it must
not be presented to unsuspecting users or used on a network without
explicit authorization.
8. DNS behavior
The DNS server listens on port 53:
DNSServer dnsServer;
During the main loop, dnsServer.processNextRequest() processes
incoming DNS requests.

In the captive-portal demonstration, DNS requests are directed to the
ESP32's access-point address. A browser may then reach the local web
interface rather than the destination it originally requested.
9. Access-point state
The web interface contains controls that change the access-point state.

When the hotspot state is started, the source:
- Stops the DNS server.
- Disconnects the existing soft access point.
- Reconfigures the access-point IP.
- Starts an open access point using the selected network's SSID.
- Restarts DNS redirection.
When stopped, it restores the original hard-coded access-point name and
password.

This behavior imitates the SSID of a selected network. It does not
make the ESP32 a legitimate extension of that network and does not
establish that the selected router is under the operator's control. This
is the rogue-access-point/Evil-Twin concept demonstrated by the source.
10. Deauthentication code: important limitation
The source includes:
- A deauthing_active state flag.
- A deauthentication-frame byte array.
- The selected network's BSSID copied into the frame.
- A channel-selection condition.
However, the raw transmission calls are commented out:
// wifi_send_pkt_freedom(...)
The code therefore constructs and prints frame data but does not
transmit those frames using the commented calls. The presence of a
button or state flag alone does not mean that a deauthentication attack
is functioning.

Deauthentication can disrupt connectivity. Do not enable or adapt it
against networks or clients without explicit authorization.
11. Main loop: loop()
The loop performs several recurring tasks:
1. Processes DNS requests.
2. Handles incoming HTTP client requests.
3. Checks the deauthentication state and its timer.
4. Runs a Wi-Fi scan approximately every 15 seconds.
5. Prints whether the ESP32 station interface is connected
   approximately every 2 seconds.
The serial messages GOOD and BAD indicate the station connection
status according to WiFi.status().
12. Source-code observations and limitations
This README describes the uploaded source as written. The following
points are important when reviewing it:
- The web page accepts a password and the source stores it in
  _tryPassword.
- A successful connection causes the source to print the submitted
  password in a serial message. This is unsafe credential handling.
- The code's deauthentication transmission calls are commented out, so
  the shown deauthentication section does not send those frames
  through those calls.
- handleResult() treats a successful station connection as evidence
  that the submitted password worked; connection state alone should be
  interpreted carefully.
- The source uses a fixed access-point password in the firmware.
  Hard-coded credentials should not be reused in deployed devices.
- The network scan stores a maximum of 16 results; additional networks
  are not retained.
- The source should be checked against the exact ESP32 Arduino core
  version being used. API availability and behavior can vary by
  version.
- The HTML is assembled using scanned network names and other strings.
  Untrusted values should be HTML-escaped before being inserted into a
  page.
- The program does not implement a secure administrative login,
  transport encryption, or safe credential storage.
13. Safe lab and learning objectives
Use this project to study the underlying concepts without collecting
real credentials or disrupting third-party networks:
1. Understand AP + station mode on ESP32.
2. Learn how DNS requests are handled by a local DNS server.
3. Understand HTTP request routing and web-server handlers.
4. Observe how Wi-Fi scans report SSID, BSSID, and channel information.
5. Study the difference between a legitimate access point and a
   look-alike rogue access point.
6. Inspect management-frame concepts using controlled packet captures.
7. Learn how captive portals can be abused and how users can identify
   suspicious login prompts.
8. Practice defensive monitoring and incident response.
For demonstrations, use a dedicated test SSID and test devices that you
own. Use a dummy form field rather than a real Wi-Fi password, and
remove password retention and serial logging. Keep the lab isolated from
production and public networks.
14. Defensive guidance
- Do not enter Wi-Fi credentials into unexpected browser pop-ups or
  firmware-update pages.
- Verify the access-point identity and security settings before
  connecting.
- Prefer WPA2-AES or WPA3 with a strong, unique passphrase.
- Disable WPS if it is not needed.
- Keep router and client firmware updated.
- Monitor for unexpected duplicate SSIDs and suspicious access points.
- Use wireless intrusion detection where appropriate.
- Treat unexpected disconnections and repeated authentication prompts
  as potential warning signs.
15. Project summary
This ESP32 source combines wireless network discovery, a local access
point, DNS redirection, and an HTTP captive-portal-style interface. It
also contains code associated with rogue access points and
deauthentication, although the shown raw deauthentication transmission
calls are commented out.

The most important lesson is that Wi-Fi security is not only about
encryption: users must also be able to identify the network they are
joining and recognize when a login prompt is untrusted.

Use only in an isolated, authorized security lab.