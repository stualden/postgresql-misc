
This is a tutorial for installing a PostgreSQL server in a jail on TrueNAS.  My original motivation was to give myself a SQL-capable back-end that I could use for my data science lessons.  I now use it for that as well as to maintain various databases I need (like personal finances, etc.)  

As of this writing, I'm working with the following versions of the software:

* TrueNAS:		12.0-U4  (latest stable version)
* PostgreSQL:  	13.3     (latest non-beta FreeBSD version available in the repository)
	
#### Prequisites
* A working TrueNAS installation, connected to the Internet, with some spare disk space available

#### Follow these 8 steps to install PostgreSQL in an iocage jail:

1. Carve out some space to hold your databases - create a new dataset dedicated to that purpose.

	* Go to **Storage / Pools**
	* Select your pool, click on the 3 dots and select [Add Dataset]
		* Name:  (whatever - I use postgresql)
		* Comment:  (whatever)
		* Keep all remaining defaults and click **Save**

2. Set up a new jail for running the PostgreSQL server:

	- Go to **Jails**
		- (If you don't have any jails yet, select a pool for jail storage)
	- Click on **Add** and use the sequence of Basic pages
		- Name:  (whatever - I use postgresql)
		- Jail type:  (I use basejail - see [here](https://www.truenas.com/community/threads/iocage-jail-type-base-jail-vs-clone-which-to-choose.82639/) for great discussion of jail types)
		- Release:  might as well use the latest (12.2-RELEASE as of this writing)
		- Networking: (whatever you like; I use DHCP Autoconfigure IPv4, VNET, and Berkeley PacketFilter, and then my router hands the jail a fixed lease.)
		- Check Auto-start so the jail will always  be running
		- Keep all remaining defaults and click **Save**

3. Set up a mount point so the jail from step 2 can access the dataset in step 1.

	- Select the jail, click the pull down at the far right, click on **Mount Points**
	- For Source, choose the dataset you created in step 1.
	- For Destination, choose something within the jail (I use /mnt/postgresql)
	- Click **Save**

4. Install PostgreSQL in the jail, using the `pkg` command in the shell.

	* Click on the jail, select the pulldown, and click on **Shell**
	* At the command prompt, type `pkg` and hit enter
		* If not yet installed, answer y to install pkg
	* Install the following packages:
	
		`pkg install postgresql13-server-13.3_1`
		
		`pkg install postgresql13-contrib-13.3`
		
		`pkg install postgresql13-docs-13.3`
	
	Note:  A search on the [ports page](https://www.freebsd.org/cgi/ports.cgi) for something like 'postgresql13' will show you what you're installing.

	While you're at it, install a basic editor for some editing that you'll be doing shortly.  I use nano:

	`pkg install nano`

5. While still at the jail shell prompt, adjust some settings, set up proper ownership, and initialize and run postgresql

	* Make sure postgresql starts up when jail is launched; at the prompt type
	
		`sysrc postgresql_enable=YES`

		(Note that this just adds a line **postgresql_enable="YES"** to the end of the /etc/rc.conf file.)		

	* Tell postgres the location to hold databases by referring to the mount the point you set up in step 3

		`sysrc postgresql_data=/mnt/postgresql`
		
		(Similarly, this adds a line **postgresql_data="/mnt/postgresql"** to /etc/rc.conf)

	* Change user and group ownership of that location to the **postgres** user and group:

		`chown -R postgres:postgres /mnt/postgresql`
		
	(Note that the **postgres** user and group were created when you installed postgres.)


6) Initialize postgres and ensure that you can access it.

	* Initialize postgres

		`service postgresql initdb`

	* Make modifications to permit access of the server from other hosts. By default, all connections will be refused. To change that, we need to edit **postgresql.conf** and **pg_hba.conf**, files that were produced from the initdb.

		`nano /mnt/postgresql/postgresql.conf`
		
		Where  you see **listen_addresses**, add this line if it's okay for any host to try to connect:

		`listen_addresses = '*'`

		Save and exit (e.g., with nano, use Ctrl+x, y, enter).  Then edit pg_hba.conf:

		`nano /mnt/postgresql/pg_hba.conf`

		To the end of the file, add a line to specify how hosts on your LAN can access postgresql.  For example, if your LAN is 192.168.1.0/24 and you don't require any special security, you would add a line like
		
		`host all all 192.168.1.0/24 trust`

		Save and exit.

	* Now start up postgresql and make sure it's running

		`service postgresql start`
		
		`service postgresql status`

7. Create a database and a user so you can test access to the system
			
	`su postgres`  (take over as the postgres superuser; prompt will change to $)

	`createuser --interactive`

	Provide a name (I use **stu**) and make it a superuser.  Then create a new database 'new_db':

	`createdb new_db`

	Launch an interactive SQL session to configure new user permissions
	
	`psql`  (prompt will change to postgres=#)

	`ALTER USER stu WITH ENCRYPTED PASSWORD 'INSERT_YOUR_PASSWORD_HERE';`  (need the quotes around the password)

	`GRANT ALL PRIVILEGES ON DATABASE new_db TO stu;`

	Exit psql and your postgres shell
	
	`\q`
	
	`exit`

	Restart the service
	
	`service postgresql restart`

8. Connect to the server using your favorite tool (e.g. pgAdmin or DBeaver), using the jail's IP address and the default port of 5432. Check to make sure you can access the new_db database.

