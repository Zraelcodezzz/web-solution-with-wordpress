# Web-solution-with-wordpress

We'll be prepare storage infrastructure on two linux servers and implement a web solution using wordpress.WordPress is a popular open-source content management system (CMS) that allows users to create, manage, and publish websites easily. It was initially designed for blogging but has since evolved into a versatile platform that supports various types of websites, including e-commerce stores, portfolios, business websites, and more.



This Project has two parts:

1. Configure storage subsystem for Web and Database servers: Gain hands-on experience with disks, partitions, and volumes on Linux.
2. Install WordPress and connect it to a remote MySQL database server: Solidify your skills in deploying web and database tiers of a web solution.


We'll be implmenting this web solution on a three tier architecture.

***

A three-tier architecture is a software structure that divides an application into three layers:

Presentation Tier: This is the user interface where people interact with the application (like a website or app).

Application Tier: This layer handles the business logic and processes user requests. It acts as a bridge between the presentation and data tiers.

Data Tier: This layer stores and retrieves data, usually from a database


### 3-Tier Setup

 **Client**: Your personal laptop or PC, serving as the client.
 **Web Server**: An EC2 Linux Server (RedHat OS) where WordPress will be installed.
 **Database Server**: An EC2 Linux Server (RedHat OS) designated to function as the database (DB) server.

We'll be making use of redhat distro for our ec2 instance in this project


![image](https://github.com/user-attachments/assets/b19a6b69-c10c-4e44-bfbe-9eff252f4294)

***

# STEP 1 : PREPARE THE WEB SERVER

i. After launching our web server ec2 we'll be creating three volumes in the same availabilty zone as our web server each of 10GIB

ii. Next we attach the EBS volumes one after the other to the web server ec2

![image](https://github.com/user-attachments/assets/fb791334-ab47-47a0-ac11-6fa8005f7670)



iii. Then we use `lsblk` to inspect block devices attached to the server.

               lsblk

![image](https://github.com/user-attachments/assets/09026773-0b62-47f3-b4d9-37e43ba4a204)

iv. Use 'df-h' to see all mounts and free space on server


                df-h 


 v.  Use gdsisk utility to create a single partition on each of the three disks.


         sudo gdisk /dev/<volumeid>

         
![image](https://github.com/user-attachments/assets/f4254b8c-5619-47bb-aaf3-b6b36dbc1992)

 vi. Use `lsbklk` to view the newly configured partitions

 vii.Install `lvm2` package using and run `lvmdiskscan` to  check for available partitions

            sudo yum install lvm2
             sudo lvmdiskscan
viii. Use `pvcreate` to match each of the three(3) disks as physical volumes

        sudo pvcreate /dev/xvdb1
        sudo pvcreate /dev/xvdc1
         sudo pvcreate /dev/xvdd1

Then check if created successfully using 
              
               sudo pvs

ix.  Use `vgcreate` utility to add all physical volumes to a volume group using

               sudo vgs

x. Now let's create Logical Volumes (LVs):

Create two logical volumes:

          apps-lv using half of the VG size.
          logs-lv using the remaining space.
          sudo lvcreate -n apps-lv -L 14G webdata-vg
           sudo lvcreate -n logs-lv -L 14G webdata-vg
           Verify the entire setup
           
Then verify the setup using

         sudo vgdisplay -v #view complete setup - VG, PV, and LV
         sudo lsblk

xi. Next we format the logical volumes  with ext4 system using `mkfs.ext4`

       sudo mkfs -t ext4 /dev/webdata-vg/apps-lv
      sudo mkfs -t ext4 /dev/webdata-vg/logs-lv


xii.  Now we create directories for our website storage and for backup of our logdata

           sudo mkdir -p /var/www/html       
           
          sudo mkdir -p /home/recovery/logs


 xiii.     We mount /var/www/html on apps-lv volume

              sudo mount /dev/webdata-vg/apps-lv /var/www/html/

 xiv.          Before mounting our log files we use `rsyn` to backup our logiles

                sudo rsync -av /var/log/ /home/recovery/logs/

  xv.           Then we mount using

                 sudo mount /dev/webdata-vg/logs-lv /var/log

   xvi.         Restore the log files back into the /var/log directory
   
                 sudo rsync -av /home/recovery/logs/ /var/log       

     Update /etc/fstab:
xvii.          Update /etc/fstab so that the mount configuration persists after a server reboot:
                         
                         sudo blkid
                         sudo vi /etc/fstab
                         
  _Note_: the UUID of our apps and logs...ensure to remove the quotes when adding the UUUID into the file
xviii. Test configuration and reload daemon

                sudo mount -a
   
               sudo systemctl daemon-reload
  Test setup using   
  
                 df -h

  ![image](https://github.com/user-attachments/assets/5ef3ccd5-67e8-47a6-95f9-c0b22ed811cc)       
  
***

# STEP 2: PREPARE THE DATABASE SERVER

Launch another Redhat EC2 instance 'DB server' and repeat the procedure we did for Web server but instead of apps-lv create db-lv and mount it on /db directory instead of /var/www/html


![image](https://github.com/user-attachments/assets/a52aae83-bda3-4fbd-9a2c-a31100a20812)


# STEP 3: INSTALL WORDPRESS ON WEB SERVER EC2

i. Firstly, update the system 

         sudo yum -y update

  ![image](https://github.com/user-attachments/assets/23ecaf49-38f8-4ea1-a6bf-55b81536eeda)


ii. Next we install Apache and PHP

         sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json

iii.  Start and enable apache

            sudo systemctl enable httpd
            sudo systemctl start httpd

 iv. Install PHP and it's dependencies

          sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
          sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
          sudo yum module list php
           sudo yum module reset php
           sudo yum module enable php:remi-7.4
           sudo yum install php php-opcache php-gd php-curl php-mysqlnd
            sudo systemctl start php-fpm 
             sudo systemctl enable php-fpm 
             sudo setsebool -P httpd_execmem 1


![image](https://github.com/user-attachments/assets/bd0b4921-5a37-452e-b717-f7cfb3798de6)


![image](https://github.com/user-attachments/assets/6c4e1612-7e27-4cf6-aeb0-f477134e06d2)


 v.  Restart Apache

                   sudo systemctl restart httpd

vi. Next we download and configure ecpress

    mkdir wordpress
    cd wordpress
     sudo wget http://wordpress.org/latest.tar.gz
      sudo tar xzvf latest.tar.gz
      sudo rm -rf latest.tar.gz
      sudo cp wordpress/wp-config-sample.php wordpress/wp-config.php
      sudo cp -R wordpress /var/www/html/


      
![image](https://github.com/user-attachments/assets/65437ce6-cef1-4a96-b1fe-c9b11891a31e)

***

 # STEP 3: INSTALL MYySQL on DB Server EC2     

i. Update the system then install mysql  

       sudo yum update
       sudo yum install mysql-server

  ![image](https://github.com/user-attachments/assets/898d1deb-941f-4c32-abf6-ab7498ce53a8)


![image](https://github.com/user-attachments/assets/bc822ae7-af9d-4144-b7ac-22985719c56a)


 ii. Start and update the service

         sudo systemctl start mysqld
         sudo systemctl enable mysqld

 ![image](https://github.com/user-attachments/assets/662954ff-b22d-46f1-b058-80d181f56681)

  # STEP 4: CONFIFURE MYSQL FOR WORDPRESS

Create a DATABASE and User with permissions.

      sudo mysql
     CREATE DATABASE wordpress;
      CREATE USER 'myuser'@'<Web-Server-Private-IP-Address>' IDENTIFIED BY 'mypass';
       GRANT ALL ON wordpress.* TO 'myuser'@'<Web-Server-Private-IP-Address>';
        FLUSH PRIVILEGES;
        SHOW DATABASES;
         exit

         
![image](https://github.com/user-attachments/assets/2989aace-8d65-496e-9499-77f7841b25da)

         
#  STEP 5: CONFIGURE WORDPRESS TO CONNECT TO REMOTE DATABASE

i. Go to your webserver terminal and install mysql


    sudo yum install mysql
    sudo mysql -u myuser -p -h <DB-Server-Private-IP-Address>
    
 ii. Verify the connection using 


     SHOW DATABASES;

     ![image](https://github.com/user-attachments/assets/c31979f3-7e80-44d5-b00a-a4e19c19d595)

 iii.  Open port 3306 for MySQL DB server allowing access from Web Server Private IP 



![image](https://github.com/user-attachments/assets/374c824a-9b03-4324-a817-65d4d30a3316)


 iv.  Navigate to wordpress directory in web server instance  


 v.  We want to configure wordpress to use our host database now open the file and edit 

     sudo vi wp-config.php

![image](https://github.com/user-attachments/assets/c4cc5c3e-11f3-43e9-897b-6bef764b447e)


vi.  Open up browser and access  wordpress 

     http://<Web-Server-Public-IP-Address>/wordpress/




![image](https://github.com/user-attachments/assets/ef61d4e4-28a8-436c-a4c9-f77d0fe6e30c)

![image](https://github.com/user-attachments/assets/3256fe8e-763a-42f8-8967-6c354a21fda9)

![image](https://github.com/user-attachments/assets/13c10b4c-c18d-4035-83ed-0055d38cdf76)


Cheers!


