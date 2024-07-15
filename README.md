#!/bin/bash

# Backup the current /etc/network/interfaces file
cp /etc/network/interfaces /etc/network/interfaces.bak

# Start the new configuration file
echo "# This file was automatically generated" > /etc/network/interfaces
echo "auto lo" >> /etc/network/interfaces
echo "iface lo inet loopback" >> /etc/network/interfaces
echo "" >> /etc/network/interfaces

# Loop through each network interface
for iface in $(ip -o link show | awk -F': ' '{print $2}' | grep -v lo); do
    echo "auto $iface" >> /etc/network/interfaces

    # Check if the interface uses DHCP
    dhcp=$(grep -q "dhcp" <<< $(ip addr show $iface) && echo "yes" || echo "no")

    if [ "$dhcp" = "yes" ]; then
        echo "iface $iface inet dhcp" >> /etc/network/interfaces
    else
        # Get static IP information
        ip_address=$(ip -o -4 addr show $iface | awk '{print $4}')
        gateway=$(ip route show default | grep $iface | awk '{print $3}')
        dns_servers=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}')

        echo "iface $iface inet static" >> /etc/network/interfaces
        echo "    address $ip_address" >> /etc/network/interfaces
        echo "    gateway $gateway" >> /etc/network/interfaces

        if [ -n "$dns_servers" ]; then
            echo "    dns-nameservers $dns_servers" >> /etc/network/interfaces
        fi
    fi

    echo "" >> /etc/network/interfaces
done

echo "New /etc/network/interfaces file generated."
