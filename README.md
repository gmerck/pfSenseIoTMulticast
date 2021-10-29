# pfSense IoT Multicast
A walkthrough of configuring pfSense with Avahi and PIMD for multicast to use with casting devices where displaying devices are on an IOT network and user devices are on LAN


# Install required packages
Install Avahi and PIMD packages from the Package Manager

# Step 1: Service Setup
## *Services > Avahi*:

<p>

\**Note: Do not enable yet\**
> - Action: set to `Allow Interfaces`
> - Interfaces: ctrl+select interfaces to listen on (`LAN` and `IOT`)
> - Uncheck: Disable support for IPv4
> - Check: Disable support for IPv6
> - Check: Enable Reflection
> - Publishing: Enable all publishing
> - Advanced settings: no change
> - **`Save`**

</p>

## *Services > PIMD*:
### *PIMD > General*:
<p>

\**Note: Do not enable yet\**
> - Default Bind: `Bind to All`
> - Log Level: `Debug` for troubleshooting/startup, otherwise Warning
> - CARP: `none`
> - General Settings: *Leave all blank*
> - Threshold Type: `Default`
> - **`Save`**
</p>

### *PIMD > Interfaces* tab:

<p>

> - **`+Add`**
> - *Select interface to exclude (start with `WAN`)*
> - **`Save`**
  > - *Repeat for IOT and any additional interfaces*
</p>

### *PIMD > BSR Candidates* tab:

> - **`+Add`**
> - Interface: `default`
> - **`Save`**
</p>

### *PIMD > RP Candidates* tab:
</p>

> - **`+Add`**
> - NO CHANGE - Leave at Interface: `default`
> - **`Save`**
</p>

### *PIMD > RP Addresses*:
<p>

> - NO CHANGE - do not add any entries here
</p>

# Step 2: Firewall Alias setup
## *Firewall > Aliases*:
### *Aliases > IP*:
<p>

Adding the full multicast range...
> - **`+Add`**
> - Name: `MulticastRange_Full`
> - Type: `Network(s)`
> - Network of FQDN: `224.0.0.0` `/3`
> - Description: `Multicast IP range from 224.0.0.1-239.255.255.255`
> - **`Save`**
</p>

<p>

Adding the devices that will be displaying media, like Chromecast, Roku, etc.
> - **`+Add`**
> - Name: `CastingDevices`
> - Type: `Host(s)`
> - Network of FQDN: `<IP of cast device>`
> - Description: `<Description of device>`
> - **`+Add Host`** and repeat for each device you will be viewing content on
> - **`Save`**
</p>

<p>

Additional alias for allowing access anywhere **but** private networks for IOT or other guest networks used later
> - **`+Add`**
> - Name: `Private_Networks`
> - Type: `Network(s)`
> - Network of FQDN: `<Full IP pool of subnet on private interface>`
> - Description: `<Name of network pools or interfaces>`
> - Repeat for each private network
> - **`Save`**
</p>

<p>

### Aliases>Ports::
Add the ports commonly used in multicast advertising to one alias
> - **`+Add`** (Hint: use import button and copy/pasta the list below)
> - Name: `CastingPorts`
> - Type: `Port(s)`
> - Enter each of the below ports: (Covers most all Chromecast, Roku, Sonos devices)\ 
 
    5353
    1900
    8008
    8009
    8443
    5556
    5558
>    
> - **`Save`**
</p>
<br><br>

# Step 3: Firewall Rule Setup
## *Firewall > Rules*:
### *Rules > Floating*:
<p>

> - **`+Add`**
> - Action: `Pass`
> - Uncheck: `Disabled`
> - Check: `Quick`
> - Interface: `<Select same networks as Avahi setup above - networks to incude in multicast>`
> - Direction: `any`
> - Address family: `IPv4`
> - Protocol: `IGMP`
> - Source: `any`
> - Destination: `Single host or alias` : `MulticastRange_Full` alias
> - Log: *Checked to start, can be disabled*
> - Description: `Allow <networks> IGMP to multicast address range`
> - `SHOW ADVANCED`
> - Checked: `Allow IP Options`
> - **`Save`**
</p>

### *Rules > LAN*:
<p>

*Note: The below repeated rule may be possible with a single floating rule using direction "Out"*

> - **`+Add`**
> - Action: `Pass`
> - Uncheck: `Disabled`
> - Interface: `LAN` (repeat for other interface(s)
> - Address family: `IPv4`
> - Protocol: `UDP`
> - Source: `any`
> - Destination: `Single host or alias` : `MulticastRange_Full` alias
> - Port(s): `(other)`  `CastingPorts` alias
> - Log: *Check to start, can be disabled*
> - Description: `Allow <networks> UDP to multicast address range`
> - `SHOW ADVANCED`
> - Checked: `Allow IP Options`
> - **`Save`**

 ^^Repeat the above for IOT network(s)^^

</p>



### Add the following rules in top down order in the **IOT** network

<p>

> ### Allow devices to the Multicast Range of IPs
> - **`+Add`** above (top)
> - Action: `Pass`
> - Interface: `<Select IOT network>`
> - Address family: `IPv4`
> - Protocol: `UDP`
> - Source: `any`
> - Port: `any`
> - Destination: `Single host or alias` : `MulticastRange_Full` alias
> - Destination Port(s): `(other)` : `CastingPorts` alias
> - Log: *Check to start, can be disabled later*
> - Description: `Allow casting devices to advertise mDNS or SSDP`
> - **`Save`**

</p>

<p>

> ### Allow output devices to the LAN
> - **`+Add`** above 
> - Action: `Pass`
> - Interface: `<Select IOT network>`
> - Address family: `IPv4`
> - Protocol: `any`
> - Source: `Single host or alias` : `CastingDevices` alias
> - Destination: `LAN net`
> - Log: *Check to start, can be disabled*
> - Description: `Allow only casting devices to LAN`
> - `SHOW ADVANCED`
> - Checked: `Allow IP Options`
> - **`Save`**
> - Move to 2nd down in list order

</p>

<p>

> ### Allow IOT network devices to go anwhere but the LAN
> #### *Recommended additional rule to block IOT devices besides your output devices from LAN and other networks*
> - **`+Add`** 
> - Action: `Pass`
> - Interface: `<Select IOT network>`
> - Address family: `IPv4`
> - Protocol: `any`
> - Source: `IOT net`
> - Destination: CHECK: `invert match`  `Single host or alias` : `Private_Networks` alias
> - Log: *Check to start, can be disabled*
> - Description: `Allow IOT anywhere but private networks`
> - **`Save`**
> - Move to 3rd down in list order
</p>


# Step 4: Configuration File Modification - Fix TTL
## *Diagnostics > Edit File*

<p>
Modifying the filter.inc config file

> - `Diagnostics`>`Edit File`
> - `Browse` to: `/etc/inc/filter.inc`
> - Find under "function filter_generate_scrubbing() {...
> - Directly under **\$scrubrules = "";** paste the following lines and edit them accordingly. Be sure to maintain indent at same level of "$scrubrules"
>   - Hint: Ctrl+F and search for

    $scrubrules = "";

##########COPY BELOW##########

            /* The following 2 lines were added to fix TTL of 1 in multicast */
                $scrubrules .= "scrub in on \$<NameOfIOTInterface - IOT> inet proto udp from <IP network of IOT in CIDR format - 192.168../24> to 239.255.255.250 port 1900 min-ttl 2 {$scrubnodf} {$scrubrnid}     {$mssclamp} fragment reassemble\n";
                $scrubrules .= "scrub in on \$<NameOfLANInterface - LAN> inet proto udp from <IP network of LAN in CIDR format - 192.168../24> to 239.255.255.250 port 1900 min-ttl 2 {$scrubnodf} {$scrubrnid}     {$mssclamp} fragment reassemble\n";

##########COPY ABOVE##########

> - *Be sure to update information between <> components and remove the <> symbols.*
> - Click **`Save`**.
</p>

<p>

# Step 5: Enabling Services
## *Services > Avahi*:
> - Check: `Enable`
> - **`Save`**

## *Services > PIMD*:
> - Check: `Enable`
> - **`Save`**

Reboot to refresh rules, states, and new filter.inc file. NOTE: The above lines may need to be re-added after a version update!!

Once the network comes back up, monitor mobile devices for new destinations in the cast list.
</p>

## Enjoy, and let me know how it worked for you, or if you have any recommended changes in the comments.
