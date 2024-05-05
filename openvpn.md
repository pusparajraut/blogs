 

Introduction

A Virtual Private Network (VPN) encrypts all network traffic, masking the users and protecting them from untrusted networks. It can provide a secure connection to a company network, bypass geo-restrictions, and allow you to surf the web using public Wi-Fi networks while keeping your data private.

OpenVPN is a fully-featured, open-source Secure Socket Layer (SSL) VPN solution.

Prerequisites

A CentOS 7 or CentOS 8 server
A user account with root (sudo) access
Access to the command line/terminal window
A domain or subdomain that resolves to your server
A client machine from which you will connect to the OpenVPN server
Step 1: Install OpenVPN

1. Update the CentOS repositories and packages by running:

yum update -y

2. You cannot download the OpenVPN package from the default CentOS repositories. However, OpenVPN is available in the Extra Packages for Enterprise Linux (EPEL) repository. To enable the EPEL repository, run the command:

yum install epel-release -y



3. Update the repositories again:

yum update -y

4. You can now install OpenVPN with the command:

yum install -y openvpn



Step 2: Install Easy RSA

The next step is to build a Public Key Infrastructure (PKI). To do this, you need to install easy RSA, a CLI utility for creating and managing a PKI Certificate Authority (CA).

Easy RSA helps you set up an internal certificate authority (CA) and generate SSL key pairs to secure the VPN connections.

1. To download the easy RSA package, use the wget command. If you don’t have wget on your CenOS system, install it by running:

yum install -y wget

2. At the time of writing, the latest version of the CLI utility is 3.0.8, which we will download. To use another version, check out easy RSA’s release page on GitHub.

wget https://github.com/OpenVPN/easy-rsa/archive/v3.0.8.tar.gz




3. Next, extract the downloaded archive:

tar -xf v3.0.8.tar.gz

4. Create and move into a new openvpn directory:

cd /etc/openvpn/

5. Then, create a subdirectory easy-rsa under the path /etc/openvpn:

mkdir /etc/openvpn/easy-rsa

6. Move the extracted directory into /etc/openvpn/easy-rsa:

mv /root/easy-rsa-3.0.8 /etc/openvpn/easy-rsa

To check whether you have successfully moved everything from the easy-rsa-3.0.8 directory, move into easy-rsa with cd /etc/openvpn/easy-rsa and list the content with ls. You should see a list of files and folders, as in the image below.






Step 3: Configure OpenVPN

Once you have installed OpenVPN and Easy RSA, you can move on to configuring the OpenVPN server.

The instructions in this section help you set up the basic configuration. You can alter it according to your needs.

Before running any of the commands, make sure to return to the root directory. To do so, type cd in the terminal window and hit Enter.

1. The first step is to copy the sample server.conf file from OpenVPN’s documentation directory:

cp /usr/share/doc/openvpn-2.4.9/sample/sample-config-files/server.conf /etc/openvpn

If you cannot find the OpenVPN sample configuration file, search for its location using the find command:

find / -name server.conf

2. Then, open the copied configuration file with a text editor of your choice:

vi etc/openvpn/server.conf

The command opens the sample OpenVPN config file. The comments in the file begin with a hashtag # or a semicolon ;.




3. To set up the basic configuration, you need to uncomment the following lines by removing the semicolons.

topology subnet (makes the OpenVPN installation function as a subnetwork)
push "redirect-gateway def1 bypass-dhcp" (instructs the client to redirect traffic through the OpenVPN server)
push "dhcp-option DNS 208.67.222.222" (uses an OpenDNS resolver to connect to OpenVPN)
push "dhcp-option DNS 208.67.220.220" (uses an OpenDNS resolver to connect to OpenVPN)
user nobody (runs OpenVPN with no privileges)
group nobody (runs OpenVPN with no privileges)
4. Then, generate a static encryption key to enable TLS authentication. To do that, locate the line tls-auth ta.key 0 and comment it by adding ; in front of it. Then, add a new line under it:

tls-crypt myvpn.tlsauth




Note: The configuration file specifies which DNS servers to use to connect to OpenVPN. By default, it is set to use OpenDNS resolvers, which is how we left it. Alternatively, you can change it to different DNS resolvers by modifying the push "dhcp-option DNS 208.67.222.222" and push "dhcp-option DNS 208.67.220.220" lines.

5. Save and exit the configuration file.

6. Finally, generate the static encryption key specified in the file with the command:

openvpn --genkey --secret /etc/openvpn/myvpn.tlsauth

Note: Check out our guide on how to install OpenVPN on Docker.

Step 4: Generate Keys and Certificates

1. Create a vars configuration file using vars.example stored in the /easy-rsa/easyrsa3 directory. Move into the mentioned directory with:

cd /etc/openvpn/easy-rsa/easyrsa3

2. You can list the contents using the ls command to check whether you have the vars.example file.




3. Copy the sample file vars.example under the name vars:

cp vars.example vars

If you list the files in the directory again, you should have a separate vars file that you can use to configure Easy RSA.





4. Open the vars file in a text editor of your choice:

vi vars

5. Scroll through the file and find the lines listed below.

#set_var EASYRSA_REQ_COUNTRY "US"

#set_var EASYRSA_REQ_PROVINCE "California"

#set_var EASYRSA_REQ_CITY "San Francisco"

#set_var EASYRSA_REQ_ORG "Copyleft Certificate Co"

#set_var EASYRSA_REQ_EMAIL "me@example.net"

#set_var EASYRSA_REQ_OU "My Organizational Unit"

6. Uncomment the lines by removing # and replace the default values with your information.

7. Then, find the line specifying the KEY_NAME and change it to "server":

export KEY_NAME="server"

8. Finally, change KEY_CN to the domain or subdomain that resolves to your server.

export KEY_CN=openvpn.yourdomain.com

9. Save and close the file.

10. Clean up any previous keys and generate the certificate authority:

./easyrsa clean-all




11. Now, you can move on to building the certificate authority with the build-ca script. Run the command:

./easyrsa build-ca

You will be asked to set a CA Key Passphrase and a common name for your CA.




Note: To skip password authentication each time you sign your certificates, you can use the ./easyrsa build-ca nopass command.

12. Create a key and certificate for the server:

./easyrsa build-server-full server




13. Next, generate a Diffie-Hellman key exchange file by running:

./easyrsa gen-dh




14. You also need a certificate for each client. Generate them on the server and then copy them on the client machine.

With the following command, we create a certificate and key for client1. You can modify the command by using a name of your choice.

./easyrsa build-client-full client1




15. Once you have generated the keys and certificates, copy them from pki into the openvpn directory. To do so, navigate to the pki directory by running:

cd /etc/openvpn/easy-rsa/easyrsa3/pki

You need to copy four files in total:

ca.crt
dh.pem
ca.key
server.key
The first two files (ca.crt and dh.pem) are stored in the pki directory, while ca.key and server.key are in a subdirectory pki/private.




Therefore, copy ca.crt and dh.pem into the openvpn directory first:

cp ca.crt dh.pem /etc/openvpn

Then, move into the subdirectory private, and copy ca.key and server.key by running:

cd private

cp ca.key server.key/etc/openvpn

Step 5: Firewall and Routing Configuration

Set Firewall Rules

1. Start by checking your active firewalld zone:

firewall-cmd --get-active-zones

The output will show your firewalld zone. In the example below, it is public.





2. Add the openvpn service to the list of services firewalld allows within the active zone. The active zone in our example is public. If your active zone is trusted, modify the command accordingly.

firewall-cmd --zone=public --add-service openvpn

3. Next, make the settings above permanent by running the command:

firewall-cmd --zone=public --add-service openvpn --permanent

4. To check whether the openvpn service was added use:

firewall-cmd --list-services --zone=public





5. Then, add a masquerade to the runtime instance:

firewall-cmd --add-masquerade

6. And make it permanent:

firewall-cmd --add-masquerade --permanent

7. Verify the masquerade was added by running:

firewall-cmd --query-masquerade

The output should respond with yes.




Routing the Configuration

Once you have completed the steps above, move on to routing to your OpenVPN subnet.

1. Create a variable that represents the primary network interface used by your server. In the command below, the variable is named VAR. However, you can create a variable under the name of your choice.

VAR=$(ip route get 208.67.222.222 | awk 'NR==1 {print $(NF-2)}')

2. Next, permanently add the routing rule using the variable created above:

firewall-cmd --permanent --direct --passthrough ipv4 -t nat -A POSTROUTING -s 10.8.0.0/24 -o $VAR -j MASQUERADE

3. Reload firewalld for the changes to take place:

firewall-cmd --reload

4. Move on to routing all web traffic from the client to the server’s IP address by enabling IP forwarding. Open the sysctl.conf file:

vi /etc/sysctl.conf

5. Add the following line at the top of the file:

net.ipv4.ip_forward = 1

6. Finally, restart the service:

systemctl restart network.service

Step 6: Start OpenVPN

1. To start the OpenVPN service, run the command:

systemctl -f start openvpn@server.service

2. Then, enable it to start up at boot by running:

systemctl -f enable openvpn@server.service

3. Verify the service is active with:

systemctl status openvpn@server.service

The output should respond that the OpenVPN service for the server is active (running).

Step 7: Configure a OpenVPN Client

With everything set up on the OpenVPN server, you can configure your client machine and connect it to the server.

As mentioned in Step 4, each client machine needs to have local copies of the CA certificate, client key, SSL certificate, and the encryption key.

1. Find and copy the following files from the server to the client machine:

/etc/openvpn/easy-rsa/easyrsa3/pki/ca.crt
/etc/openvpn/easy-rsa/easyrsa3/pki/client.crt
/etc/openvpn/easy-rsa/easyrsa3/pki/private/client.key
/etc/openvpn/myvpn.tlsauth
2. Then, create a configuration file for the OpenVPN client under the name client.ovpn on the client machine:

vi client.ovpn

3. Add the following content to the file:

client

tls-client

ca /path/to/ca.crt

cert /path/to/client.crt

key /path/to/client.key

tls-crypt /path/to/myvpn.tlsauth

remote-cert-eku "TLS Web Client Authentication"

proto udp

remote your_server_ip 1194 udp

dev tun

topology subnet

pull

user nobody

group nobody

Make sure to replace the bolded parts with your respected values.




4. Save and close the file.

Step 8: Connect a Client to OpenVPN

The instructions on how to connect to OpenVPN differ depending on your client machine’s operating system.

For Linux Users

To connect to OpenVPN, run the command:

openvpn --config /path/to/client.ovpn

For Windows Users

1. First, copy the client.ovpn configuration file in the C:Program FilesOpenVPNconfig directory.

2. Download and install the OpenVPN application. You can find the latest build on the OpenVPN Community Downloads page. Once you have installed the application, launch OpenVPN.

3. Right-click the OpenVPN system tray icon and select Connect. To perform this task, you need administrative privileges.

For macOS Users

You can connect to OpenVPN from a macOS system using Tunnelblick (an open-source graphic user interface for OpenVPN on OS X and macOS).

Before launching Tunnelblick, make sure to store the client.ovpn configuration file in the ~/Library/Application Support/Tunnelblick/Configurations directory.

 

 
