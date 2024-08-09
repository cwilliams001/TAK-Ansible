# TAK Server Setup with Ansible

This repository contains an Ansible playbook for automating the setup of a TAK (Team Awareness Kit) Server. It handles the installation of required packages, configuration of certificates, and initial setup of the TAK Server.

## Prerequisites

1. Ansible installed on your control node
2. Target Ubuntu or Debian-based system(s) where you want to install TAK Server
3. SSH access to the target system(s)
4. TAK Server .deb package file

## Directory Structure

```
.
├── cert-metadata.sh.j2
├── inventory.ini
├── takserver_5.2-RELEASE16_all.deb
└── tak-setup.yml
```

- `cert-metadata.sh.j2`: Jinja2 template for certificate metadata
- `inventory.ini`: Inventory file listing your target hosts
- `takserver_5.2-RELEASE16_all.deb`: TAK Server Debian package file
- `tak-setup.yml`: Main Ansible playbook for TAK Server setup

## Configuration

1. Edit the `inventory.ini` file to include the IP addresses or hostnames of your target systems.

2. Review and modify the variables in the `tak-setup.yml` playbook as needed. Key variables include:

   - `tak_version`: Version of the TAK Server package
   - `ca_common_name`: Common name for the Root CA
   - `intermediate_ca_name`: Common name for the Intermediate CA
   - `server_cert_name`: Name for the server certificate
   - `admin_cert_name`: Name for the admin certificate

3. Ensure the TAK Server .deb package file is in the same directory as the playbook.

## Usage

To run the playbook:

```bash
ansible-playbook -i inventory.ini tak-setup.yml
```

## What the Playbook Does

1. Installs required packages (nano, gnupg, OpenJDK 17)
2. Configures PAM limits
3. Sets up PostgreSQL repository and installs PostgreSQL and PostGIS
4. Installs TAK Server from the .deb package
5. Configures certificates (Root CA, Intermediate CA, Server Certificate, Admin Certificate)
6. Modifies TAK Server configuration
7. Enables and starts the TAK Server service
8. Configures UFW (Uncomplicated Firewall)

## Post-Installation

After running the playbook:

1. The admin certificate (`webadmin.p12` by default) will be copied to the home directory of the Ansible user on the target system.
2. The intermediate CA truststore will also be copied to the home directory.
3. You can use these files to set up client connections to your TAK Server.

## Notes

- This playbook assumes a Debian-based system (tested on Ubuntu).
- Ensure you have adequate permissions to perform all operations on the target system.
- Review and adjust firewall rules as needed for your specific environment.



## TODO: Configure Certificate AutoEnrollment

After the initial setup, you may want to configure Certificate AutoEnrollment. This process allows the TAK Server to issue certificates to clients upon successful authentication. Follow these steps:

1. Identify the signing truststore:
   ```
   ls -l /opt/tak/certs/files/*-signing.jks
   ```

2. Configure the Marti dashboard:
   - Navigate to Configuration > Security and Authentication
   - Under Security Configuration, select Edit Security
   - Check "Enable Certificate Enrollment" and select "TAK Server CA"
   - Enter the following:
     - Signing Keystore File: `certs/files/<CACommonName>-signing.jks`
     - Signing Keystore Password: (default is `atakatak` or as defined in `cert-metadata.sh`)
     - Validity Days: (enter desired certificate validity period)

3. Modify the CoreConfig.xml file:
   ```
   sudo su tak
   cd /opt/tak
   vi CoreConfig.xml
   ```
   
   - Locate the `certificateSigning` element
   - Modify `nameEntry` values as needed
   - Add `CAkey` and `CAcertificate` attributes to `TAKServerCAConfig`:
     ```xml
     <TAKServerCAConfig keystore="JKS" keystoreFile="certs/files/TAK-ID-CA-01-signing.jks" keystorePass="atakatak" validityDays="30" signatureAlg="SHA256WithRSA" CAkey="/opt/tak/certs/files/<CAcommonName>" CAcertificate="/opt/tak/certs/files/<CAcommonName>"/>
     ```
   - Add `x509checkRevocation="true"` to the `<auth>` element

4. Validate the configuration:
   ```
   ./validateConfig.sh
   ```

5. Restart the TAK Server service:
   ```
   exit
   sudo systemctl restart takserver.service
   ```

6. Monitor the log for errors:
   ```
   tail -f /opt/tak/logs/takserver-messaging.log
   ```

Note: Be cautious when adding multiple OU nameEntry values, as there is a known bug with iTAK/iTAK Tracker in this scenario.

Remember to adjust these steps as needed for your specific environment and requirements.

## Troubleshooting

If you encounter issues:

1. Check the Ansible output for error messages.
2. Review TAK Server logs on the target system (`/opt/tak/logs/`).
3. Ensure all prerequisites are met and the TAK Server .deb file is correct.

## Security Considerations

- Change default passwords and certificate passphrases in a production environment.
- Review and adjust firewall rules as needed.
- Ensure proper file permissions for sensitive files (certificates, keys).

## Contributing

Feel free to submit issues or pull requests if you have suggestions for improvements or encounter any problems.

