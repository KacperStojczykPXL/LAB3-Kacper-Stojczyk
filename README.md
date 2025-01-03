# LAB3-Kacper-Stojczyk
DevNet opdracht Networks expert

## Part 1: Install the virtual lab environment

### VM

De bijgeleverde OVA VM is geïmporteerd in VirtualBox. Na het opstarten moeten de packet tracer EULA's geaccepteerd worden. Hierna is de VM klaar voor gebruik.

### Accounts

De volgende accounts zijn nodig:
Github, Cisco DevNet

Hierna kan er naar de volgende stap gegaan worden.

## Part 2: Install the CSR1000v VM

Dit proces is op het begin hetzelfde als de vorige stap. De OVA is geïmporteerd in VirtualBox. Hoewel moet hier ook de CD drive ISO aanpassen naar de CSR1000v ISO. Dit kan gedaan worden door de volgende stappen te volgen:

In VirtualBox: Gezocht naar VM-instellingen > Opslag > CD-apparaat (eerste CD Drive in de lijst).
Huidige ISO vervangen door decsr1000v-universalk9.16.09.05.iso.

In het geval van virtualbox moet nog in de Network Manager gecontroleerd worden dat de vxboxnet0 adapter met ipv4 192.168.56.1/24 is ingesteld. Dit was bij mij het geval dus hier heb ik verder niks aangepast.

Na het opstarten van de VM begint de eerste configuratie. Dit kan enkele minuten duren bij de eerste keer opstarten. 

Hierna verkrijg je de terminal nadat je enter drukt.

```bash
CSR1kv>
```
Hierna kan je enable uitvoeren. Er is geen password.

Verder kunnen we verifiëren of de IP die eerder vermeld was in orde is:

```bash
CSR1kv#show ip interface brief
``` 

We starten de VM uit de eerste stap op en verbinden deze met de CSR1000v VM. Dit kan gedaan worden door de terminal te openen en het ping commando te gebruiken.

Daarna leggen we een SSH verbinding met de CSR1000v VM vanuit de DEVASC VM. Met het password cisco123!, kunnen we direct inloggen met het standaard ssh commando.

```bash
ssh cisco@IP
```

Door het ip van de CSR1000v VM in de browser met https te gebruiken, kunnen we de webui interface bereiken. De browser gaat hoogstwaarschijnlijk zeggen dat het insecure is, maar dit is normaal. Hier kunnen we inloggen met de credentials cisco/cisco123!.
Op de host machine kan dezelfde webui op deze manier ook bereikt worden.

## Part 3: Python Network Automation with NETMIKO

## Part 4: Explore YANG Models

We halen de nodige yang file(ietf-interfaces.yang) op van Github(https://github.com/YangModels/yang/tree/main/vendor/cisco/xe/1693), deze zetten we op de DEVASC VM. 

Deze file wordt overgezet in de devnet-src/pyang folder op de VM.
Hier kan je verifiëren of pyang geinstalleerd is met het volgende commando:

```bash
pyang -v
``` 
Met het command pyang -f tree ietf-interfaces.yang kan je de structuur van de yang file bekijken op een leesbaardere manier.

## Part 5: Use NETCONF to Access an IOS XE Device

Start de CSR1000v VM op en verbind met deze vauit de DEVASC VM met ssh.
Met het commando show platform software yang-management process status kan je verifiëren of de netconf ssh daemon actief is.

```bash
ncssh: Running
```
We verbreken de ssh verbinding en maken een nieuwe verbinding met netconf ssh.

```bash
ssh -s cisco@IP -p 830 -s netconf
```
Hier verschijnt een xml hello message.
We plakken de volgende code in de ssh sessie om een reply te doen.

```xml
<hello xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<capabilities>
<capability>urn:ietf:params:netconf:base:1.0</capability>
</capabilities>
</hello>
]]>]]>
```
Als volgende sturen we een get RPC message:

```xml
<rpc message-id="103" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<get>
<filter>
<interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces"/>
</filter>
</get>
</rpc>
]]>]]>
```
Hier krijgen we een xml reply met de interfaces.
De reply kan leesbaarder gemaakt worden met een prettify tool.
De sessie kan afgesloten worden met het volgende commando:

```xml
<rpc message-id="9999999" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
<close-session />
</rpc>
``` 
Dit is niet user friendly manier, waardoor we nu gaan overschakelen naar ncclient.
We gebruiken de volgende script:

```python
from ncclient import manager
m = manager.connect(
host="192.168.56.147",
port=830,
username="cisco",
password="cisco123!",
hostkey_verify=False
)
```
We kunnen dit uitvoeren met het volgende commando:

```bash
python3 ncclient-netconf.py
```
In de CSR1000v VM kunnen we AUTH PASSED zien in de terminal.
We voegen de volgende code toe aan het script om de capabilities te printen:

```python
print("#Supported Capabilities (YANG models):")
for capability in m.server_capabilities:
    print(capability)
```
We commenten dit stukje code uit om de output te verkleinen. We voegen het volgende toe:

```python
netconf_reply = m.get_config(source="running")
print(netconf_reply)
```
Zoals eerder kunnen we de output leesbaarder maken met een prettify tool.
Dit kan ook gedaan worden met de volgende code:

```python
import xml.dom.minidom

print(xml.dom.minidom.parseString(netconf_reply.xml).toprettyxml())
```
Dit kan nog een filter bovenop krijgen:

```python
netconf_filter = """
<filter>
<native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native" />
</filter>
"""
netconf_reply = m.get_config(source="running", filter=netconf_filter)
print(xml.dom.minidom.parseString(netconf_reply.xml).toprettyxml())
```
We kunnen met een config variable de hostname veranderen:
```python
netconf_hostname = """
<config>
<native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
<hostname>NEWHOSTNAME</hostname>
</native>
</config>
"""
netconf_reply = m.edit_config(target="running", config=netconf_hostname)
print(xml.dom.minidom.parseString(netconf_reply.xml).toprettyxml())
```
Als volgende gaan we een loopback interface toevoegen:

```python
netconf_loopback = """
<config>
<native xmlns="http://cisco.com/ns/yang/Cisco-IOS-XE-native">
<interface>
<Loopback>
<name>1</name>
<description>My first NETCONF loopback</description>
<ip>
<address>
<primary>
<address>10.1.1.1</address>
<mask>255.255.255.0</mask>
</primary>
</address>
</ip>
</Loopback>
</interface>
</native>
</config>
"""
netconf_reply = m.edit_config(target="running", config=netconf_loopback)
print(xml.dom.minidom.parseString(netconf_reply.xml).toprettyxml())
```
Op de CSR1000v VM kunnen we de loopback interface zien met het commando show ip interface brief.
Als we een nieuwe loopback interface proberen toe te voegen met dezelfde ip, krijgen we een error.(Device refused one or more commands).

## Part 6: Use RESTCONF to Access an IOS XE Device

Om RESTCONF te kunnen gebruiken, moeten we nginx aanzetten op de CSR1000v VM.

```bash
configure terminal
ip http secure-server
ip http authentication local
``` 
Verder gaan we overschakelen naar Postman. Als eerste moet SSL verification uitgezet worden in de settings. Hierna kunnen we een nieuwe request maken met de volgende settings:

GET request naar de ip van de CSR1000v VM met /restconf.
Basic auth: cisco/cisco123!
JSON als datatype, met Headers: Key:  Content-Type, Value: application/yang-data+json.

Als volgende willen we info verkrijgen over een specifieke interface:
/restconf/data/ietf-interfaces:interfaces/interface=GigabitEthernet1
We passen de ip aan van de interface binnen de CSR1000v VM.
Wanneer je de request opnieuw uitvoert, krijg je de nieuwe info van de interface.

Als volgende gaan we een nieuwe loopback interface toevoegen met een PUT request.
/restconf/data/ietf-interfaces:interfaces/interface=Loopback1
Als body gebruiken we de volgende JSON:

```json
{
"ietf-interfaces:interface": {
"name": "Loopback1",
"description": "My first RESTCONF loopback",
"type": "iana-if-type:softwareLoopback",
"enabled": true,
"ietf-ip:ipv4": {
"address": [
{
"ip": "10.1.1.1",
"netmask": "255.255.255.0"
}
]
},
"ietf-ip:ipv6": {}
}
}
```

Indien de request succesvol is, krijg je een 201 Created response.
Je kan de nieuwe loopback interface zien met het commando show ip interface brief.

Voor het volgend stukje schakele we over naar Python.

We maken een nieuw script 'restconf-get' aan met de volgende code:

```python
import json
import requests
requests.packages.urllib3.disable_warnings()
api_url = "https://192.168.56.101/restconf/data/ietf-interfaces:interfaces"
headers = { "Accept": "application/yang-data+json",
"Content-type":"application/yang-data+json"
}
basicauth = ("cisco", "cisco123!")
resp = requests.get(api_url, auth=basicauth, headers=headers, verify=False)
print(resp)
response_json = resp.json()
print(response_json)
print(json.dumps(response_json, indent=4))
``` 
Hiermee kunnen we de interfaces zien in JSON formaat, al onmiddelijk geprettified.

Als volgende gaan we een nieuwe loopback interface toevoegen met een PUT request.

```python
import json
import requests
requests.packages.urllib3.disable_warnings()
api_url = "https://192.168.56.101/restconf/data/ietf-
interfaces:interfaces/interface=Loopback2"
headers = { "Accept": "application/yang-data+json",
"Content-type":"application/yang-data+json"
}
basicauth = ("cisco", "cisco123!")
yangConfig = {
"ietf-interfaces:interface": {
"name": "Loopback2",
"description": "My second RESTCONF loopback",
"type": "iana-if-type:softwareLoopback",
"enabled": True,
"ietf-ip:ipv4": {
"address": [
{
"ip": "10.2.1.1",
"netmask": "255.255.255.0"
}
]
},
"ietf-ip:ipv6": {}
}
}
resp = requests.put(api_url, data=json.dumps(yangConfig), auth=basicauth,
headers=headers, verify=False)
if(resp.status_code >= 200 and resp.status_code <= 299):
print("STATUS OK: {}".format(resp.status_code))
else:
print('Error. Status Code: {} \nError message:
{}'.format(resp.status_code,resp.json()))
```

Als we dit script uitvoeren, krijgen we een 201 Created response.
We kunnen de nieuwe loopback interface zien met het commando show ip interface brief.

### Part 7: Getting started with NETCONF/YANG - Part 1

Cisco ondersteunt NETCONF/YANG vanaf IOS XE 16.3. Hiermee kun je netwerkapparaten beheren via programmeren.

### NETCONF inschakelen
Voer op een Cisco-switch het volgende commando uit:

```bash
3850-remote#conf t  
3850-remote(config)#netconf-yang
```

### Transport testen
NETCONF gebruikt SSH (poort 830). Maak verbinding via:

```bash
ssh -p 830 gebruiker@IP-adres
```

Na inloggen ontvang je een <hello>-bericht met NETCONF-capabilities.

### RPC-oproepen
Stuur een hello-response en gebruik daarna RPC's zoals get-config om de configuratie op te halen:

```xml
<rpc xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="101">
  <get-config>
    <source>
      <running/>
    </source>
  </get-config>
</rpc>]]>]]>
```

### NETCONF-tools
Gebruik tools zoals netconf-console voor eenvoudiger interactie:

```bash
netconf-console.py --host=IP --port=830 --hello
```
Of filter specifieke gegevens, bijvoorbeeld interfaces:

```bash
netconf-console.py --host=IP --get-config -x "interfaces/interface[name='GigabitEthernet1/0/1']"
```
### Gestructureerde data

NETCONF gebruikt XML, wat hiërarchisch en eenvoudig door software te verwerken is. Filters beperken de output tot relevante fragmenten.

## Part 8: Getting started with NETCONF/YANG - Part 2

### Vereenvoudiging met ncc

ncc (netconf client) is een tool om NETCONF/YANG eenvoudiger te leren. Download het op GitHub(github.com/CiscoDevNet/ncc).

### Installatie (Linux/MacOS)

```bash
git clone https://github.com/CiscoDevNet/ncc.git
cd ncc
virtualenv v && . v/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
./ncc.py --host 10.10.6.2 --username sdn --password wachtwoord --capabilities
```

### Gestructureerde data met YANG

Waarom YANG?
YANG biedt eenvoudiger beheer dan SNMP, vooral bij het onderscheiden van configuratie- en operationele data. Het gebruikt containers, lists en leaves zoals <interfaces> en <name>.

Voorbeeld: Interfaces configuratie ophalen

```bash
./ncc.py --host 10.10.6.2 --username sdn --password wachtwoord --snippets ./snippets-xe --get-running --named-filter ietf-intf
```

Operationele data (oper-data)

Gebruik een sjabloon om polling te activeren:

```bash
./ncc.py --host 10.10.6.2 --do-edits 00-oper-data-enable
./ncc.py --host 10.10.6.2 --get-oper -x /interfaces-state
```

### Modules uitbreiden
Met tools zoals Pyang kunnen YANG-uitbreidingen, zoals ipv4-attributen, worden gevisualiseerd.