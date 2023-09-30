# Xbox_Nat_Helper
  Beta Version 1.0
Introducing... Xbox NAT Helper 1.0 – Your All-in-One Solution for Effortless Xbox Network Setup!

Are you tired of the complexities of configuring your Xbox for online gaming and streaming? Say goodbye to those lengthy, confusing setups, and say hello to simplicity with Xbox NAT Helper 1.0!

Check it out:

1. User-Friendly Interface: Xbox NAT Helper 1.0 boasts an easy-to-use interface powered by Zenity dialogs, making the setup process a breeze!

Interactively configure Wi-Fi and Ethernet interfaces, Bridge IP, Xbox IP, DNS servers, and more...
Firewall and FTP Configuration: Worried about firewall rules and FTP setup? Xbox NAT Helper can handle that too, automatically configuring them to work seamlessly with your Xbox.

Depends on Zenity for Menus A
nd IPtables for firewall/FTP config. 
Created with help from ChatGPT.




#!/bin/bash

# Function to check if running as root
check_root() {
    if [[ $EUID -ne 0 ]]; then
        if [[ -z $SUDO_EXECUTED ]]; then
            zenity --password --title="Enter Root Password" --text="This script requires root privileges. Please enter the sudo password." | sudo -S "$0" "$@"
            exit $?
        else
            zenity --error --text="Script is already running with sudo privileges. Aborting."
            exit 1
        fi
    fi
}

# Check if running as root
check_root "$@"


# Function to cleanup and kill the animation
cleanup() {
    # Kill the animation process
    kill "$animation_pid" &>/dev/null
    clear
}




# Function to animate "Press any key to continue..."
animate_press_any_key() {
    while true; do
        for frame in "█▀▀█ ░▀░ █▀▀█ ░▀░ █▀▀█ █░█" "█░░█ ▀█▀ █▄▄█ ▀█▀ █▄▄█ █▀█" "▀▀▀▀ ▀▀▀ ▀░░▀ ▀▀▀ ▀░░▀ ▀░▀"; do
            clear
            cat << "EOF"
    )           )     )        
 ( /(    (   ( /(  ( /(    
 )\()) ( )\  )\()) )\())   
((_)\  )((_)((_)\ ((_)\   
__((_)((_)_   ((_)__((_)   
\ \/ / | _ ) / _ \\ \/ /  
 >  <  | _ \| (_) |>  <       
/_/\)\ |___/(\___/(_/\_\   
     __   _   _____                                
  /\ \ \ /_\ /__   \   
 /  \/ ///_\\  / /\/  
/ /\  //  _  \/ /     
\_\ \/ \_/ \_/\/   
              _                    
  /\  /\ ___ | | _ __    ___  _ __ 
 / /_/ // _ \| || '_ \  / _ \| '__|
/ __  /|  __/| || |_) ||  __/| |   
\/ /_/  \___||_|| .__/  \___||_|   
                |_|                               
EOF
            echo -e "\e[92m$frame\e[0m"
            echo -e "\e[93mBeta version 1.0 made by Jordan D\e[0m"
            sleep 0.5
        done
    done
}

# Start the "Press any key to continue..." animation in the background
animate_press_any_key &

# Store the PID of the animation process
animation_pid=$!



# Configuration file
CONFIG_FILE="xboxnat.config"

# Load configuration values from the file if it exists
if [ -f "$CONFIG_FILE" ]; then
    source "$CONFIG_FILE"
fi

# Function to view the configuration
view_configuration() {
    zenity --text-info --title "Current Configuration" --filename "$CONFIG_FILE" --width 400 --height 300
}


# Function to revert changes
revert_changes() {
    # Disable IP forwarding
    sudo sh -c "echo 0 > /proc/sys/net/ipv4/ip_forward"

    # Remove NAT rules
    sudo iptables -t nat -F

    # Check if the bridge interface exists before attempting to delete it
    if [ -n "$BRIDGE_INTERFACE" ] && [ -e "/sys/class/net/$BRIDGE_INTERFACE" ]; then
        sudo ip link set dev "$BRIDGE_INTERFACE" down
        sudo ip link delete "$BRIDGE_INTERFACE" type bridge
    fi

    # Restart networking
    sudo systemctl restart networking
}



# Function to handle the setup menu
setup_menu() {
    while true; do
        choice=$(zenity --list --title "Xbox NAT Helper Setup Menu" \
            --column "Option" \
            "Setup NAT" \
            "View Config" \
            "Firewall/FTP Rules" \
            "Exit" \
            --width 246 --height 238)
        # Check if the user clicked "Cancel" or closed the dialog
        if [ $? -ne 0 ]; then
            zenity --info --text="Canceling setup..." --timeout=5
            cleanup
            exit 0
        fi
        case "$choice" in
            "Setup NAT")
                setup_nat_and_bridge
                result=$?
                if [ $result -eq 0 ]; then
revert_changes
                    zenity --info --text="Removing Bridge Returning to setup...Wait..." --timeout=5
                          else
                    zenity --error --text="NAT setup failed." --timeout=5
                fi
                ;;
            "View Config")
                view_configuration
                ;;
            "Firewall/FTP Rules")
                configure_firewall_and_ftp
                zenity --info --text="Firewall/FTP rules configured." --timeout=5
                ;;
            "Exit")
                cleanup
                exit 0
                ;;
        esac
    done
}


# Function to set up NAT and create a bridge interface with progress tracking
setup_nat_and_bridge() {
       if [ $? -eq 1 ]; then
        revert_changes
        return  # Exit the function and return to the setup menu
    fi
    
    # Set variables interactively
    set_variables_interactively
    
    # Enable IP forwarding
    if ! sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"; then
        zenity --error --title="Permission Denied" --text="Permission denied. Please enter your password to enable IP forwarding." --timeout=5
        if ! sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"; then
            zenity --error --title="Failed to Enable IP Forwarding" --text="Failed to enable IP forwarding. Please check your privileges and try again." --timeout=5
            return 1
        fi
    fi

    # Create a temporary file to hold progress data
    progress_file=$(mktemp)

    # Initialize progress to 0%
    (
        echo "10"  # Initial progress value for bridge setup
        echo "Setting up the bridge interface..."
        sleep 3  # Simulate bridge setup delay
        echo "20"  # Update progress to 20%
    ) | zenity --progress --title="Setting up NAT and Bridge" --text="Setting up the bridge interface..." --percentage=0 --auto-close

    # Create the bridge interface
    if ! sudo ip link add name "$BRIDGE_INTERFACE" type bridge; then
        zenity --error --title="Failed to Create Bridge" --text="Failed to create the bridge interface. Please check your configuration and try again." --timeout=5
        rm "$progress_file"
        return 0
    fi

    ((progress = 20))  # Update progress to 20%
    echo "$progress" > "$progress_file"

    # Add Ethernet to the bridge
    (
        echo "Adding Ethernet interface to the bridge..."
        sleep 2
    ) | zenity --progress --title="Setting up NAT and Bridge" --text="Adding Ethernet interface to the bridge..." --percentage=$progress --auto-close

    if ! sudo ip link set "$ETHERNET_INTERFACE" master "$BRIDGE_INTERFACE"; then
        zenity --error --title="Failed to Add Ethernet to Bridge" --text="Failed to add the Ethernet interface to the bridge. Please check your configuration and try again." --timeout=5
        rm "$progress_file"
        return 1
    fi

    ((progress += 10))  # Update progress to 30%
    echo "$progress" > "$progress_file"

    if ! sudo ip addr add "$BRIDGE_IP/$BRIDGE_NETMASK" dev "$BRIDGE_INTERFACE"; then
        zenity --error --title="Failed to Add IP to Bridge" --text="Failed to add an IP address to the bridge interface. Please check your configuration and try again." --timeout=5
        rm "$progress_file"
        return 1
    fi

    (
        echo "Bringing up the bridge interface..."
        sleep 2
    ) | zenity --progress --title="Setting up NAT and Bridge" --text="Bringing up the bridge interface..." --percentage=$progress --auto-close

    if ! sudo ip link set dev "$BRIDGE_INTERFACE" up; then
        zenity --error --title="Failed to Bring Up Bridge" --text="Failed to bring up the bridge interface. Please check your configuration and try again." --timeout=5
        rm "$progress_file"
        handle_setup  # This will handle the setup cancellation
        return 1
    fi

    ((progress += 10))  # Update progress to 50%
    echo "$progress" > "$progress_file"

    # Set up NAT for outbound traffic on the Wi-Fi interface
    (
        echo "Setting up NAT for outbound traffic..."
        sleep 2
    ) | zenity --progress --title="Setting up NAT and Bridge" --text="Setting up NAT for outbound traffic..." --percentage=$progress --auto-close

    if ! sudo iptables -t nat -A POSTROUTING -o "$WIFI_INTERFACE" -j MASQUERADE; then
        zenity --error --title="Failed to Set Up NAT" --text="Failed to set up NAT for outbound traffic. Please check your configuration and try again." --timeout=2
        rm "$progress_file"
        return 1
    fi

    (
        echo "Completing the setup..."
        sleep 2
    ) | zenity --progress --title="Setting up NAT and Bridge" --text="Completing the setup..." --percentage=80 --auto-close

    # Simulate a delay for the remaining setup
    sleep 1

    # Update progress to 100%
    echo "100" > "$progress_file"

    # Display the completion message for a few seconds
    sleep 1
    zenity --info --title="Setup Complete" --text="NAT set up successfully.\n\nHere Is Static Info for Xbox \n\nXbox IP Address: $XBOX_IP\nGateway IP: $BRIDGE_IP\nDNS Server 1: $DNS_SERVER1\nDNS Server 2: $DNS_SERVER2" --ok-label="OK"

    # Remove the temporary progress file
    rm "$progress_file"
    cleanup
    exit 0
}

# Function to save configuration data to the file
save_configuration() {
    echo "WIFI_INTERFACE=\"$WIFI_INTERFACE\"" > "$CONFIG_FILE"
    echo "ETHERNET_INTERFACE=\"$ETHERNET_INTERFACE\"" >> "$CONFIG_FILE"
    echo "BRIDGE_INTERFACE=\"$BRIDGE_INTERFACE\"" >> "$CONFIG_FILE"
    echo "BRIDGE_IP=\"$BRIDGE_IP\"" >> "$CONFIG_FILE"
    echo "BRIDGE_NETMASK=\"$BRIDGE_NETMASK\"" >> "$CONFIG_FILE"
    echo "XBOX_IP=\"$XBOX_IP\"" >> "$CONFIG_FILE"
    echo "DNS_SERVER1=\"$DNS_SERVER1\"" >> "$CONFIG_FILE"
    echo "DNS_SERVER2=\"$DNS_SERVER2\"" >> "$CONFIG_FILE"
}

# Function to simulate "Loading Setup menu and initialization" with a progress bar
simulate_loading_setup() {
    (
        # Simulate initialization steps
        echo "10"  # Update progress to 10%
        sleep 1

        echo "30"  # Update progress to 30%
        sleep 1

        echo "60"  # Update progress to 60%
        sleep 1

        echo "90"  # Update progress to 90%
        sleep 1

        echo "100"  # Update progress to 100%
    ) | zenity --progress \
        --title="Loading Setup" \
        --text="Loading Setup, Please Wait..." \
        --percentage=0 \
        --auto-close

    if [ $? -eq 0 ]; then
        setup_menu
    fi
}


# Function to set variables interactively using Zenity
set_variables_interactively() {
    # Select Wi-Fi Interface
    WIFI_INTERFACE=$(zenity --list --title "Select Wi-Fi Interface" --text "Choose your Wi-Fi interface:" --column "Device" $(ip -o link show | awk -F': ' '{print $2}') --height 400 --width 300 --cancel-label="Cancel")
    
    # Check if the user clicked Cancel
    if [ $? -eq 1 ]; then
        zenity --info --title "Setup Canceled" --text "Setup was canceled. Returning to the setup menu." --timeout=5
        setup_menu
        return
    fi

    # Select Ethernet Interface
    ETHERNET_INTERFACE=$(zenity --list --title "Select Ethernet Interface" --text "Choose your Ethernet interface:" --column "Device" $(ip -o link show | awk -F': ' '{print $2}') --height 400 --width 300 --cancel-label="Cancel")

    # Check if the user clicked Cancel
    if [ $? -eq 1 ]; then
        zenity --info --title "Setup Canceled" --text "Setup was canceled. Returning to the setup menu." --timeout=5
      setup_menu
        return
    fi

    # Set the bridge interface to "br0"
    BRIDGE_INTERFACE="br0"

    # Bridge IP and Netmask
    BRIDGE_IP=$(zenity --entry --title "Bridge IP Address" --text "Enter Bridge IP Address (e.g., 192.168.1.1):" --entry-text "$BRIDGE_IP" --cancel-label="Cancel")

    # Check if the user clicked Cancel
    if [ $? -eq 1 ]; then
        zenity --info --title "Setup Canceled" --text "Setup was canceled. Returning to the setup menu." --timeout=5
       setup_menu
        return
    fi

    # Default netmask (preset as 255.255.255.0)
    BRIDGE_NETMASK="255.255.255.0"

    # Check if the user wants to change the netmask
    netmask_input=$(zenity --entry --title "Bridge Netmask" --text "Enter Bridge Netmask (e.g., 255.255.255.0):" --entry-text "$BRIDGE_NETMASK" --cancel-label="Cancel")

    # Check if the user clicked Cancel
    if [ $? -eq 1 ]; then
        zenity --info --title "Setup Canceled" --text "Setup was canceled. Returning to the setup menu." --timeout=5
       setup_menu
        return
    fi

    # Validate the netmask format
    if [[ "$netmask_input" =~ ^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
        BRIDGE_NETMASK="$netmask_input"
    fi

    # Xbox IP Address
    XBOX_IP=$(zenity --entry --title "Xbox IP Address" --text "Enter Xbox IP Address (e.g., 192.168.1.2):" --entry-text "$XBOX_IP" --cancel-label="Cancel")

    # Check if the user clicked Cancel
    if [ $? -eq 1 ]; then
        zenity --info --title "Setup Canceled" --text "Setup was canceled. Returning to the setup menu." --timeout=5
        setup_menu
        return
    fi

    # DNS Servers
    DNS_SERVER1=$(zenity --entry --title "DNS Server 1" --text "Enter DNS Server 1:" --entry-text "$DNS_SERVER1" --cancel-label="Cancel")

    # Check if the user clicked Cancel
    if [ $? -eq 1 ]; then
        zenity --info --title "Setup Canceled" --text "Setup was canceled. Returning to the setup menu." --timeout=5
        setup_menu
        return
    fi

    DNS_SERVER2=$(zenity --entry --title "DNS Server 2" --text "Enter DNS Server 2:" --entry-text "$DNS_SERVER2" --cancel-label="Cancel")

    # Check if the user clicked Cancel
    if [ $? -eq 1 ]; then
        zenity --info --title "Setup Canceled" --text "Setup was canceled. Returning to the setup menu." --timeout=5
        setup_menu
        return
    fi

    save_configuration
}



# Function to configure firewall rules and FTP settings
configure_firewall_and_ftp() {
    # Ask the user whether to use the default FTP port or specify a custom port
    port_choice=$(zenity --list --title="FTP Port Configuration" \
        --text="Choose FTP port configuration:" \
        --radiolist --column="" --column="Option" \
        FALSE "Use default FTP port (21)" \
        TRUE "Specify a custom port" \
        --height 200 --width 300)

    if [ "$port_choice" = "Use default FTP port (21)" ]; then
        # Use default FTP port (21)
        ftp_port=21
    else
        # Specify a custom FTP port
        custom_port=$(zenity --entry --title="Custom FTP Port" \
            --text="Enter a custom FTP port (e.g., 2121):")
        if [[ ! $custom_port =~ ^[0-9]+$ ]]; then
            zenity --error --title="Invalid Port" --text="Invalid port number. Please enter a valid numeric port." --timeout=5
            return 1
        fi
        ftp_port="$custom_port"
    fi

    # Ask the user for the static Xbox IP address (default: 192.168.1.2)
    xbox_ip=$(zenity --entry --title="Static Xbox IP Address" \
        --text="Enter the static Xbox IP address (default: 192.168.1.2):" --entry-text "192.168.1.2")
    if [[ ! $xbox_ip =~ ^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
        zenity --error --title="Invalid IP Address" --text="Invalid IP address format. Please enter a valid IP address." --timeout=5
        return 1
    fi

    # Check if NAT is already configured
    if ! iptables -t nat -C POSTROUTING -o "$WIFI_INTERFACE" -j MASQUERADE &>/dev/null; then
        # Configure FTP port forwarding using iptables
        # Delete any existing rules for FTP (port 21) and add the new rule
        sudo iptables -t nat -D PREROUTING -p tcp --dport 21 -j DNAT --to-destination "$xbox_ip:$ftp_port" &>/dev/null
        sudo iptables -t nat -A PREROUTING -p tcp --dport 21 -j DNAT --to-destination "$xbox_ip:$ftp_port"
        
        zenity --info --title="Firewall Rules Configured" --text="Firewall rules have been configured for FTP.\n\nXbox IP: $xbox_ip\nFTP Port: $ftp_port" --timeout=10
    fi
}


# Function to handle the setup process
handle_setup() {
    # Check if the bridge interface exists
    if [ $? -eq 1 ]; then
          revert_changes
        # Return to the setup menu if the bridge exists
        return
    fi
    # Continue with the setup
      
    set_variables_interactively
}


# Display the setup menu
simulate_loading_setup
setup_menu
