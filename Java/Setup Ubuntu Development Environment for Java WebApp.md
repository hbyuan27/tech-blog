Setup Ubuntu Development Environment for Java WebApp
================================================================

Tags： Dev-Env Java Ubuntu Spring-Boot

----------

For Windows users, you can download the official Ubuntu image and use VirtualBox to start a virtual machine as the host OS for the coming steps.
Don't forget to install the VirtualBox Guest Additions to enable those useful features:
 - Shared clipboard
 - Mouse pointer integration
 - Generic host/guest communication channels
 - Seamless windows
 - Shared folders
 - ...

# **Update Softwares**
Before using Ubuntu `apt-get` command to install softwares, you'd better run `sudo apt-get update` to update the source list of software packages.
If you encounter problems to connect to the main Ubuntu server, you can switch to a mirror server at "**System Setting** -> **Software & Updates**".

# **Install & Configure JDK**
- Check if JDK is already installed:

  `java -version`

  If there is no result, choose one from the candidate list to install, e.g.

  `sudo apt-get install default-jdk`

- The alternative way is directly search and install it by:

  `sudo apt-cache search jdk`

  `sudo apt-get install <your-preferred-jdk>`
- Run `java -version` again to check if the installation is successful.

- Managing Multiple Java Installations

  Run `sudo update-alternatives --config java` to get the installation list and choose one as your default java version.

- Setting the **JAVA_HOME** Environment Variable

  So that Java servers like Tomcat, or tools like Jenkins can use your installed JRE with environment variable **$JAVA_HOME**.
  Run `sudo nano /etc/environment`, at the end of file, enter:

  `JAVA_HOME="usr/lib/jvm/<your_jdk_root_path>"`

  Save(ctrl+o) and exit(ctrl+x), then reload it: `source /etc/environment`

  Now you can run `echo $JAVA_HOME` to see the path you just set.

# **Install Eclipse**
You can install it via below command:

`sudo apt-get install eclipse`

It is easy but:
 - You cannot specify the version. (Usually it uses the old version other than the latest one)
 - It is for Java only and there is no plugins within it.
 - OpenJDK will be downloaded and installed as the dependency. But for some developers they may not want to install it.

The recommended way is to download a preferred full-size package from [offical website][1]. Then extract files into /opt/ and change the access permission.

``` bash
sudo tar -zxvf eclipse-jee-neon-3-linux-gtk-x86_64.tar.gz -C /opt
sudo chown -R root:root /opt/eclipse
sudo chmod -R +r /opt/eclipse
```

Or you can download the installer instead of the full-size package from [here][2].

``` bash
sudo tar -zxvf eclipse-inst-linux64.tar.gz -C /opt
cd /opt/eclipse-installer/
sudo ./eclipse-inst
```

Follow the guide to complete the installation. (Choose /opt/ as the installation path when using sudo ./eclipse-inst)

Now you can start your eclipse by run `/opt/eclipse/eclipse -clean &`

**Shortcut for Launching** (just in case you the shortcut is missing)

If the eclipse shortcut is missing, you have to create it (Gnome menu item) manually and drag it onto taskbar or desktop.

You need to create a file named *Eclipse.desktop* in *"/usr/share/applications/"* , or in *"~/.local/share/applications/eclipse.desktop"* (for current user only):
e.g.
**gedit ~/.local/share/applications/eclipse.desktop**
``` properties
[Desktop Entry]
Type=Application
Name=Eclipse
Comment=Eclipse Integrated Development Environment
Icon=/opt/eclipse/icon.xpm
Exec=something like /opt/eclipse/eclipse
Terminal=false
Categories=Development;IDE;Java;
StartupWMClass=Eclipse
```
With execute privilege granted:

`chmod +x ~/.local/share/applications/eclipse.desktop`

Drag it to the taskbar.

# **Install Maven**

`sudo apt-get install maven`

or use `apt-cache search maven` to find the version you'd like to use.

Use `mvn -version` to check the installation.

# **Install Git**

`sudo apt-get install git`

Use `git --version` to check the installation.

# **Install PostgreSQL**
- Install Local Database

  `sudo apt-get install postgresql`

  The default user name and database name is: **postgres**

  Login and switch to the psql command window:

  `sudo -u postgres psql postgres`
  Setup a new password for default user *postgres* :
  ```
  postgres=# \password postgres
  Enternew password:
  Enter it again:
  postgres=# \q
  ```

- Create your own user and database if you don't want to use the default one
  ``` bash
  sudo -u postgres createuser -D -A -P <your_user>
  sudo -u postgres createdb -O <your_user> <your_db>
  ```

- Install Vitualized Database Management Tool:

  `sudo apt-get install phppgadmin`

  Enter `http://localhost/phppgadmin` in your browser and you can find all the details.

# **Build Spring-Boot WebApp**
 - Clone the code from [Code on GitHub][3]
 - Modify the local database connection in ***application.properties*** :
  ``` properties
  spring.datasource.url= jdbc:postgresql://localhost:5432/<your_db>
  spring.datasource.username=<your_user>
  spring.datasource.password=<your_pwd>
  spring.datasource.driverClassName=org.postgresql.Driver
  spring.jpa.hibernate.ddl-auto=create-drop
  ```
  
- Go to the project root folder, run `mvn spring-boot:run`
 - Once the application is up, open *http://localhost:8080/hello* in your browser, you will add a record in the database.
 - Send another request via *http://localhost:8080/history*, you can see all data records listed.


  [1]: http://www.eclipse.org/downloads/packages/
  [2]: https://www.eclipse.org/downloads/?
  [3]: https://github.com/hbyuan27/spring-boot-webapp-demo.git
