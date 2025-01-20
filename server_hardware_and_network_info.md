# Server hardware and network info

# Table of contents

[TOC]

# Overview

NOTE: This info is very specific to my particular server, home network setup, and ISP

# Server hardware info

- https://www.theserverstore.com/Dell-PowerEdge-R630-36-CORE-Virtualization-Server-192GB-3x-800GB-SSD-H730p_p_988.html
- https://www.google.com/url?sa=t&source=web&rct=j&opi=89978449&url=https://i.dell.com/sites/doccontent/shared-content/data-sheets/en/documents/dell-poweredge-r630-spec-sheet.pdf&ved=2ahUKEwiOmZmr_bCEAxW3FjQIHexrAh8QFnoECCYQAQ&usg=AOvVaw0GT_z0LYLwK9Rt-9iKSgBz
- https://www.google.com/url?sa=t&source=web&rct=j&opi=89978449&url=https://i.dell.com/sites/doccontent/shared-content/data-sheets/en/Documents/Dell-PowerEdge-R630-Technical-Guide-v1-6.pdf&ved=2ahUKEwiOmZmr_bCEAxW3FjQIHexrAh8QFnoECCgQAQ&usg=AOvVaw0UcLDArCKOOhUeVQuWJ_oZ
- https://www.google.com/url?sa=t&source=web&rct=j&opi=89978449&url=https://www.dell.com/support/manuals/en-us/poweredge-r630-dsms/r630_om_pub/dell-poweredge-r630-system-overview%3Fguid%3Dguid-a743b271-3784-4599-8c90-0fbca035aeb0%26lang%3Den-us&ved=2ahUKEwiO2JzZ_rCEAxXVHjQIHdIyBgwQFnoECA4QAQ&usg=AOvVaw14gsicB1V6nroCDHgMTogF
- https://www.dell.com/support/manuals/en-us/poweredge-r630-dsms/r630_om_pub/dell-poweredge-r630-system-overview?guid=guid-a743b271-3784-4599-8c90-0fbca035aeb0&lang=en-us

# Network info

## Basic network and machine IP info

- Home routers and subnets:
    - External modem/router subnet: `10.0.0.0` (XFinity model CGM4981COM modem/router - admin webpage http://10.0.0.1/)
    - Internal router subnet (all home machines): `192.168.1.0` (TP-Link Archer - admin webpage http://192.168.1.1/webpages/login.html)
    - All admin passwords in 1password
- Main development machine `192.168.1.111`
- `poweredge` server: `192.168.1.200`, connected to `eno1`, MAC `14:18:77:57:75:36`
- Router DHCP configured to assign static IPs based on MAC addresses

# Client network setup

These general steps should be performed on all client machines that will be used to access the server. Do them right away,
because later steps will depend on them (e.g., all subsequent seteps assume that the `poweredge` host resolves to the server).

## Add hosts entry to main development machine

- Add `192.168.1.200 poweredge` to `/etc/hosts`

# Dynamic DNS Raspberry PI setup

- Raspberry PI Zero 2 W
- DNS provider namecheap.com
    - https://www.namecheap.com/support/knowledgebase/article.aspx/595/11/how-do-i-enable-dynamic-dns-for-a-domain/
    - https://www.namecheap.com/support/knowledgebase/article.aspx/5/11/are-there-any-alternate-dynamic-dns-clients/
- TODO: finish setup, test, and document    
