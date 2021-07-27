# Déploiement d'un site avec Ant et  apache tomcat et apache

1. ## Creation d'un virtualhost sous apache

   Premièrement, nous allons créer un nouvel virtualhost `cours.misa.mg`. Donc, editer le fichier `/etc/hosts` et ajouter la ligne suivante

   ```bash
   127.0.1.1		cours.misa.mg
   ```

   Notre fichier `/etc/hosts` devient donc comme suit:

   ```bash
   127.0.1.1       site.univ.mg
   127.0.1.1       site.mada.mg
   # Inserer la ligne suivante
   127.0.1.1		cours.misa.mg
   
   # The following lines are desirable for IPv6 capable hosts
   ::1     ip6-localhost ip6-loopback
   
   ```

   Et puis aller dans le dossier `/etc/apache2/sites-availables` et ajouter le fichier suivant: `cours.conf`

   ```
   <VirtualHost *:80>
       ServerAdmin webmaster@localhost
       
       Servername cours.misa.mg
   
       CustomLog ${APACHE_LOG_DIR}/access-ua-sciences.log combined
       ErrorLog ${APACHE_LOG_DIR}/error.log
   
       #SSLEngine on
       #SSLOptions +FakeBasicAuth +ExportCertData +StrictRequire
   
       #SSLCertificateFile     /etc/apache2/ssl/ua-sciences.mg.crt
       #SSLCertificateKeyFile /etc/apache2/ssl/ua-sciences.mg.key
       ProxyRequests On
       ProxyPass / http://cours.misa.mg:8080/
       ProxyPassReverse / http://cours.misa.mg:8080/
   
       #<FilesMatch "\.(cgi|shtml|phtml|php)$">
       #               SSLOptions +StdEnvVars
       #</FilesMatch>
       #<Directory /usr/lib/cgi-bin>
       #               SSLOptions +StdEnvVars
       #</Directory>
           
   </VirtualHost>
   ```

   Maintenant il faut activer le module proxy et http_proxy dans apache. Taper les commandes suivantes:

   ```shell
   user@pc-user:~$ sudo a2enmod proxy
   user@pc-user:~$ sudo a2enmod proxy_http
   ```

   Et puis activer notre virtualhost avec la commande suivante:

   ```
   user@pc-user:~$ sudo a2ensite cours.conf
   ```

   Enfin restarter notre serveur apache

   ```
   user@pc-user:~$ sudo systemctl reload apache2
   ```

2. ## Création de notre projet java

   Pour commencer, créer un dossier nomé `cours` dans votre dossier home. Et crée les sous dossiers et fichiers suivantes dans le dossier cours

    ![tree](/home/raph35/Pictures/tree.png)

   Et voici les contenus de ces fichiers:

   __build.xml:__

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <!DOCTYPE xml>
   <project basedir="." default="package" name="cours">
   	
   	<property environment="env" />
   	<property name="TOMCAT_HOME" value="${env.CATALINA_HOME}" />
   	<property name="debuglevel" value="source,lines,vars" />
   	<property name="target" value="1.8" />
   	<property name="source" value="1.8" />
   	<property name="project.name" value="${ant.project.name}" />
   	<property name="project.web.dir" value="WebContent" />
   	<property name="project.src.dir" value="src" />
   	<property name="project.classes.dir" value="build/classes" />
   	<property name="project.lib.dir" value="${project.web.dir}/WEB-INF/lib" />
   	<property name="project.war" value="${project.name}.war" />
   	<property name="project.runtime.lib" value="${TOMCAT_HOME}/lib" />
   	<property name="project.deploy.location" value="${TOMCAT_HOME}/webapps" />
   
   
   	<path id="classpath.runtime">
   		<fileset dir="${project.runtime.lib}" includes="*.jar" />
   	</path>
   	<path id="classpath.lib">
   		<fileset dir="${project.lib.dir}" includes="*.jar" />
   	</path>
   	<path id="project.classpath">
   		<pathelement location="${project.classes.dir}" />
   		<path refid="classpath.runtime" />
   		<path refid="classpath.lib" />
   	</path>
   	<target depends="startTomcat" name="init">
   		<mkdir dir="${project.classes.dir}" />
   		<copy includeemptydirs="false" todir="${project.classes.dir}">
   			<fileset dir="src">
   				<exclude name="**/*.java" />
   			</fileset>
   		</copy>
   	</target>
   	<target name="clean">
   		<delete dir="${project.classes.dir}" />
   		<delete dir="${project.war}" />
   	</target>
   
   	<target depends="init" name="build" description="Compiler tous les fichiers java vers ${project.src.dir}">
   		<echo message="${project.name}: ${ant.file}" />
   		<javac debug="true" debuglevel="${debuglevel}" destdir="${project.classes.dir}" includeantruntime="false" source="${source}" target="${target}">
   			<src path="src" />
   			<classpath refid="project.classpath" />
   		</javac>
   	</target>
   	<target depends="build" name="package" description="Générer ${project.war}">
   		<war destfile="${project.war}" index="true" needxmlfile="fasle">
   			<classes dir="${project.classes.dir}" />
   			<lib dir="${project.lib.dir}" />
   			<fileset dir="${project.web.dir}">
   				<include name="**/*.*" />
   			</fileset>
   		</war>
   	</target>
   	<target depends="package" name="deploy" description="Copie ${project.war} vers ${project.deploy.location}">
   		<copy file="${project.war}" todir="${project.deploy.location}" />
   	</target>
   
   
   	<target depends="stopTomcat" name="startTomcat">
   		<exec executable="${TOMCAT_HOME}/bin/startup.sh" />
   	</target>
   	<target name="stopTomcat">
   		<exec executable="${TOMCAT_HOME}/bin/shutdown.sh" />
   	</target>
   </project>
   ```

   __index.html:__

   ```
   <!DOCTYPE html>
   <html lang="en">
   <head>
       <meta charset="UTF-8">
       <meta http-equiv="X-UA-Compatible" content="IE=edge">
       <meta name="viewport" content="width=device-width, initial-scale=1.0">
       <title>Document</title>
   </head>
   <body>
       It works!!
   </body>
   </html>
   ```

   __MANIFEST.MF:__

   ```
   Manifest-Version: 1.0
   Class-Path: 
   ```

3. ## Création d'un virtualhost dans apache tomcat

   Il faut maintenant créer un virtualhost associée à notre site dans notre serveur tomcat. Pour faire cela, il faut ajouter ses quelques lignes dans le fichier `server.xml` dans le dossier `conf` situé dans l'installation de notre apache tomcat.

   ```xml
   <Host name="cours.misa.mg"  appBase="webapps"
       unpackWARs="true" autoDeploy="true">
       <Context docBase="cours" path="" />
   
       <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
       prefix="cours_access_log" suffix=".txt"
       pattern="%h %l %u %t %; %s %b" />
   </Host>
   ```

   Ainsi, il faut copier ces lignes dans notre fichier `server.xml` à l'emplacement indiqué ci dessous:

   **server.html:**

   ```xml
   ...
   ...
           <!-- Access log processes all example.
                Documentation at: /docs/config/valve.html
                Note: The pattern used is equivalent to using pattern="common" -->
           <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
                  prefix="localhost_access_log" suffix=".txt"
                  pattern="%h %l %u %t %r %s %b" />
         </Host>
         
   		<!-- Inserer le code ci dessus ici -->
       </Engine>
     </Service>
   </Server>
   ```

   

4. ## Création de la variable d'environment `CATALINA_HOME`

   Maintenant, il faut créer la variable d'environment `CATALINA_HOME` dans notre fichier `.bashrc` situé dans votre dossier personnel. Pour cela, ajouter la ligne suivante a la fin de votre fichier `.bashrc`

   ```
   export CATALINA_HOME=/path/to/the/folder_that_containt/your_tomcat
   ```

   Remplacer `/path/to/the/folder_that_containt/your_tomcat` par le chemin où se trouve votre tomcat

   Par exemple pour le mien c'est `/opt/apache-tomcat-9.0.22`

5. ## Lancement de tomcat et deployement de notre projet dans tomcat avec ant

   Maintenant que nous avons fini de configurer tous, on va lancer notre serveur tomcat. Pour cela aller dans votre dossier où est installer tomcat et aller vers le dossier `bin` et lancer le script bash `startup.sh`

   ```
   user@pc-user:/opt/apache-tomcat-9.0.22$ cd bin
   user@pc-user:/opt/apache-tomcat-9.0.22/bin$ ./startup.sh
   ```

   Maintenant aller dans votre dossier `cours` lancer la commande `ant deploy` pour construire et déployer notre projet vers tomcat apache directement.

   ```
   user@pc-user:~$ cd cours
   user@pc-user:~$ ant deploy
   ```

   Cela va compiler le projet, créer un archive `cours.war` et copier cette archive vers le dossier `webapps` dans notre serveur tomcat.

Maintenant, ouvrer le lien suivant dans votre navigateur et notre page doit apparaitre.

Ce tuto est terminer, merci d'avoir suivi jusqu'à la fin.

**Copyright 2021 Raphael - MISA M1**

$$
\left (
\begin{array}{c c c}
1 & 2 & 6 \\
1 & 2 & 6 \\
1 & 2 & 6 \\
1 & 2 & 6 \\
\end{array}
\right)
$$
