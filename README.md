# USB Printer Network Sharing Guide
## Using CUPS and Samba on Arch Linux with Raspberry Pi

This guide explains how to share a USB printer across your local network using a Raspberry Pi running Arch Linux as a print server. This setup allows all devices on your network to access the printer, even though it lacks built-in network capabilities.

## Prerequisites
- Raspberry Pi running Arch Linux
- USB printer
- Basic knowledge of Linux command line
- Administrative access to your network

## Installation Steps

### 1. Install Required Packages
```bash
sudo pacman -S cups cups-pdf samba avahi nss-mdns
```

### 2. Configure CUPS

1. Start and enable CUPS service:
```bash
sudo systemctl start cups
sudo systemctl enable cups
```

2. Edit CUPS configuration file:
```bash
sudo nano /etc/cups/cupsd.conf
```

3. Modify the configuration to allow network access:
```conf
# Listen on all network interfaces
Port 631
Listen /run/cups/cups.sock

# Allow remote access
<Location />
  Order allow,deny
  Allow @LOCAL
</Location>

<Location /admin>
  Order allow,deny
  Allow @LOCAL
</Location>

<Location /admin/conf>
  AuthType Default
  Require user @SYSTEM
  Order allow,deny
  Allow @LOCAL
</Location>
```

### 3. Install Printer Drivers

#### Standard CUPS Drivers
Most printers work with built-in CUPS drivers. Connect your printer and check if it's detected:
```bash
lpinfo -v
```

#### Gutenprint Drivers (For Unsupported Printers)
If your printer isn't supported by CUPS:
```bash
sudo pacman -S gutenprint
```

### 4. Add Printer to CUPS

1. Access CUPS web interface:
- Open browser: `http://raspberry_pi_ip:631`
- Go to "Administration" → "Add Printer"
- Login with root credentials

2. Select your USB printer from the list

3. Choose appropriate driver:
   - Use manufacturer-specific driver if available
   - Use Gutenprint driver if standard driver not available
   - For raw printing support, use this configuration in `/etc/cups/printers.conf`:
```conf
<Printer your_printer_name>
  DeviceURI usb://YOUR_PRINTER_URI
  State Idle
  Accepting Yes
  Shared Yes
  Option raw
</Printer>
```

### 5. Configure Samba

1. Edit Samba configuration:
```bash
sudo nano /etc/samba/smb.conf
```

2. Add printer sharing configuration:
```conf
[global]
   workgroup = WORKGROUP
   printing = CUPS
   printcap name = CUPS
   load printers = yes
   cups options = raw

[printers]
   comment = All Printers
   path = /var/spool/samba
   browseable = yes
   guest ok = yes
   writable = no
   printable = yes
   create mode = 0700
```

3. Start and enable Samba:
```bash
sudo systemctl start smb
sudo systemctl enable smb
```

## Client Setup

### Windows Clients

1. Add Network Printer:
   - Open Printers & Scanners settings
   - Click "Add Printer"
   - Choose "Add a network printer"
   - Enter: `\\raspberry_pi_ip\printer_name`

2. Driver Selection:
   - If print output is corrupted, try these solutions:
     a. Use generic/universal driver instead of manufacturer-specific driver
     b. Use raw printing configuration (shown above in CUPS setup)
     c. Match driver versions between server and client

### Linux Clients

1. Add Printer:
   - Open system settings → Printers
   - Add printer with URI: `ipp://raspberry_pi_ip:631/printers/printer_name`

### macOS Clients

1. Add Printer:
   - Open System Preferences → Printers & Scanners
   - Click "+"
   - Select: `smb://raspberry_pi_ip/printer_name`

## Troubleshooting

### Common Issues

1. Printer Not Detected
   - Check USB connection
   - Verify printer is powered on
   - Run `lsusb` to confirm USB detection

2. Permission Issues
   - Add your user to required groups:
```bash
sudo usermod -aG lp,sys username
```

3. Corrupted Print Output
   - Check driver compatibility
   - Try using generic drivers
   - Configure for raw printing

4. Connection Issues
   - Verify network connectivity
   - Check firewall settings
   - Ensure CUPS and Samba services are running

## Maintenance

1. Monitor CUPS logs:
```bash
journalctl -u cups
```

2. Clear print queue:
```bash
cancel -a
```

3. Restart services if needed:
```bash
sudo systemctl restart cups smb
```

## Security Considerations

1. Limit network access to trusted IPs in CUPS configuration
2. Use strong authentication for admin access
3. Regularly update system and packages
4. Monitor printer usage through CUPS web interface

For additional support or driver updates, visit:
- CUPS Documentation: https://www.cups.org/documentation.html
- Gutenprint: http://gimp-print.sourceforge.net/
