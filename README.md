# Tableau Connector For Startree Pinot

This is the Tableau Connector for [Startree Pinot](https://startree.ai).

We are working on making this connector available in the Tableau Exchange. For now, the connector can be installed and used as follows.

## Installation & Usage
* Pinot setup requires JDK 11 and Maven 3.6

* Download the driver jars. When using Pinot 1.0.0, the driver versions below will be needed
    - async-http-client-2.12.3.jar
    - calcite-core-1.30.0.jar
    - [pinot-jdbc-client-1.0.0-hotfix-shaded.jar](https://mvnrepository.com/artifact/org.apache.pinot/pinot-jdbc-client)

* Copy the driver to the Tableau drivers directory
    - On Mac: `~/Library/Tableau/Drivers/`
    - On Windows: `C:\Program Files\Tableau\Drivers`

* Fork this repository

* Exec `mvn package` in this directory to build the connector. The `.taco` file will be in the `target` subdirectory. 

* Copy the `.taco` to the connectors directory
    - On Mac: `~/Library/Tableau/Connectors/` 
    - On Windows: `C:\Users\[user]\Documents\My Tableau Repository\Connectors`

* Launch Tableau Desktop with connector signature verification disabled.  
    - On Mac: `/Applications/Tableau\ Desktop\ <version>.app/Contents/MacOS/Tableau -DDisableVerifyConnectorPluginSignature=true` 
    - On Windows: `tableau.exe -DConnectPluginsPath=C:\tableau_connector`

* Select the connector from installed connectors

![Tableau Connectors](plugins/pinotjdbc/static/img/tableau_connector_ui.png?raw=true "Installed Connectors")

<br/>

![Apache Pinot Connection Dialog](plugins/pinotjdbc/static/img/tableau_connector_dialog.png?raw=true "Apache Pinot Connection Dialog")


![Fetch data from Pinot](plugins/pinotjdbc/static/img/tableau_tables_ui.png?raw=true "Fetch data from Pinot")

## Self-Sign the TACO file

1. Create self-signed root certificate
```shell
keytool -genkeypair -alias rootca -keystore container.jks -keyalg RSA -keysize 4096 -validity 3650 -ext BC=ca:true -storepass changeit -dname "CN=xxx Root CA, OU=security, O=xxx, L=NY, ST=NY, C=US"
keytool -exportcert -alias rootca -keystore container.jks -storepass changeit -file rootca.crt
```

2. Create sub certificate for connector
```shell
keytool -genkeypair -alias startree-connector -keystore container.jks -keyalg RSA -keysize 2048 -validity 3650 -storepass changeit -dname "CN=startree-connector, OU=data, O=xxx, L=NY, ST=NY, C=US"
keytool -certreq -alias startree-connector -keystore container.jks -file startree-connector.csr -storepass changeit
keytool -gencert -alias rootca -keystore container.jks -infile startree-connector.csr -outfile startree-connector.cer -validity 3650 -storepass changeit -ext KU=dig,keyEnc -ext EKU=codesign
```

3. Import the sub certificate to keystore
```shell
keytool -importcert -alias startree-connector -keystore container.jks -file startree-connector.cer -storepass changeit
```

4. Verify certificate
```shell
keytool -list -v -keystore container.jks -storepass changeit -alias startree-connector
```

Certificate chain length should be 2. Output sample:
```
Alias name: startree-connector
Creation date: Mar 10, 2025
Entry type: PrivateKeyEntry
Certificate chain length: 2
Certificate[1]:
Owner: CN=startree-connector, OU=data, O=xxx, L=NY, ST=NY, C=US
Issuer: CN=xxx Root CA, OU=security, O=xxx, L=NY, ST=NY, C=US
Serial number: bee7d60
Valid from: Mon Mar 10 22:40:09 AST 2025 until: Thu Mar 08 22:40:09 AST 2035
Certificate fingerprints:
         SHA1: 98:DB:CB:2D:E9:32:9E:01:FD:89:2A:67:F5:94:F6:4D:95:EC:26:25
         SHA256: 7C:CC:79:4C:BD:02:54:BB:F4:BD:59:CF:03:D7:65:F6:5E:A5:A7:19:
82:EB:19:36:1A:37:1E:DC:B2:02:74:96
Signature algorithm name: SHA384withRSA
Subject Public Key Algorithm: 2048-bit RSA key
Version: 3

......

Certificate[2]:
Owner: CN=xxx Root CA, OU=security, O=xxx, L=NY, ST=NY, C=US
Issuer: CN=xxx Root CA, OU=security, O=xxx, L=NY, ST=NY, C=US
Serial number: 34b278f8
Valid from: Mon Mar 10 22:39:38 AST 2025 until: Thu Mar 08 22:39:38 AST 2035
Certificate fingerprints:
         SHA1: DF:DB:44:FC:96:E0:D9:5E:2A:58:FE:33:8E:18:40:F6:D6:98:12:7F
         SHA256: 41:02:8E:B5:B5:0B:14:D5:EE:0F:C4:B9:48:F1:AB:4B:75:D4:BB:AD:1B:51:82:D7:3C:93:D5:08:C4:E5:90:96
Signature algorithm name: SHA384withRSA
Subject Public Key Algorithm: 4096-bit RSA key
Version: 3

......

```

5. Sign the connector
```shell
jarsigner -keystore container.jks -storepass changeit -keypass changeit -signedjar ai.startree.pinot-startree-tableau-connector-1.0-V2-signed.taco ai.startree.pinot-startree-tableau-connector-1.0-V2.taco startree-connector
```

6. Add root ca to Tableau jre key store

For MacOS:  
Go Settings -> Privacy & Security -> Full Disk Access  
Make sure the "Terminal" is on. 
```shell
cd /Applications/Tableau\ Desktop\ 2024.3.app/Contents/Plugins/jre/lib/security/
sudo keytool -importcert -alias rootca -file ~/workspace/tableau_temp/rootca.crt -keystore cacerts -storepass changeit
```

For Windows:

For Linux:

