⚠️ Please, note that this repository uses git submodules. Use flag `--recurse-submodules` when cloning it, and verify that they have been initialised.

This document helps you prepare the (mininum) settings for your LocalEGA instance.

All files are encrypted using Crypt4GH.
The master key should be stored securely.

It requires:
* a service key
* a master key
* a configuration file for the python handler: `lega.ini`
* a configuration file for docker-compose: `docker-compose.yml`
* 2 configurations file for postgres: `pg.conf` and `pg_hba.conf`

We assume you have created a local user and a group named `lega`. If not, you can do it with

    groupadd -r lega
	useradd -M -g lega -G docker lega

# Sensitive data

Update the configuration files with the proper settings.
> Hint: copy the supplied sample files and adjust the passwords, paths, etc., appropriately.  

	cp docker-compose.yml.sample           docker-compose.yml
	cp ../../src/vault/pg.conf.sample      pg.conf
	cp ../../src/vault/pg_hba.conf.sample  pg_hba.conf
	cp ../../src/handler/conf.ini.sample   lega.ini

==**Ojo!!!**==
The included message broker uses an administrator account with
`admin:secret` as `username:password`. This is up to you to update it
in your production environment.

Generate the service key with:

	ssh-keygen -t ed25519 -f service.key -C "service_key@LocalEGA"
	chown lega service.key
	chown lega service.key.pub

Note: You will get prompted for the passphrase. Save it and update
`lega.ini` accordingly, with the proper filepath and the chosen
passphrase. (it is _not_ recommended _not to use_ any passphrase).

Repeat the same for the master key:

	ssh-keygen -t ed25519 -f master.key -C "master_key@LocalEGA"
	chown lega master.key
	chown lega master.key.pub
**He puesto ega y egamaster como contraseñas rspectivamente.**
	
# Mountpoints / File system

Prepare the storage mountpoints for:
* the inbox of the users
* staging area
* the vault location
* the backup location

```bash
	# Create the directories (some with the setgid bit)
	mkdir -p data/{inbox,staging,vault,vault.bkp}

	# Change the ownership
	chown lega:lega data/{inbox,staging,vault,vault.bkp}

	# Change the access permissions
	chmod 2750 data/inbox # with the setgid bit, the `lega` user can _read_ the inbox files of each user.
	                      # Other users then the owner can't.
	chmod 700 data/staging
	chmod 750 data/vault  # lega group needs r,x in order to distribute files
	chmod 700 data/vault.bkp
```
Adjust the paths in the `docker-compose.yml` file and the `lega.ini` handler configuration.

# Container images

Create the docker images with:

	make -j3 images LEGA_UID=$(id -u lega) LEGA_GID=$(id -g lega)

# The vault database

Prepare the vault database 

	echo 'very-strong-password' > pg_vault_su_password
	**pwd in my case: pgpwd**
	chmod 600 pg_vault_su_password
	make init-vault
	
	# start the database
	docker-compose up -d vault-db
	
Update the database password for the following database users. First
use `make psql`, to connect, and then issue the following SQL
commands:

	-- To input data
	ALTER ROLE lega WITH PASSWORD 'strong-password';
	ALTER ROLE lega WITH PASSWORD 'pglega';
	** i wrote pglega **

	-- To distribute data
	ALTER ROLE distribution WITH PASSWORD 'another-strong-password';
	ALTER ROLE distribution WITH PASSWORD 'pgdist';
	** i wrote pgdist **
Update the handler `lega.ini` configuration file, with the `lega` user password from the database.

In the `pg.conf` file, update the `crypt4gh.master_seckey` secret with the hex value of the master private key.  
You can run the following python snippet to get it: (you need the `crypt4gh` package: `pip install crypt4gh`).

```python
import crypt4gh.keys

key_content = crypt4gh.keys.get_private_key("/path/to/master.key.sec", lambda: "passphrase")

print(key_content.hex())
```
**remember route now is ./master.key and pwd=legamaster**

The `pg_hba.conf` controls the network accesses to the database.  
The default supplied one is not very restrictive, and you should adjust it in your production environment.  
(For example, by enabling TLS/SSL in the `pg.conf` and restricting network CIDRs in `pg_hba.conf`).

# Instantiate the containers 

Finally, you are now ready to instantiate the containers

	# We start with the inbox, and the broker, (the vault database is already started above)
	docker-compose up -d inbox mq vault-db
	
	# We wait a bit, and check that they are up
	# And we start the handler, that connects to the broker and the vault database
	docker-compose up -d handler

You can follow along with

	docker-compose logs -f

and tear all down with

	docker-compose down -v

Note that the `mq` component will try to create a federated queue to another RabbitMQ server. In `cega` folder, you will find the necessary components to fake Central EGA, and test your local deployment in isolation.
