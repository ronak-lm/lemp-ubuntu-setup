
# LEMP Setup on Ubuntu Server

## 1. Install Nginx

Run the following commands:

```bash
sudo apt-get update
sudo apt-get install nginx
sudo ufw allow 'Nginx Full' # adds nginx to firewall
sudo ufw allow 'OpenSSH'    # adds ssh to firewall
sudo ufw enable             # enables firewall
```

Check firewall rules by typing `sudo ufw status`. It should return this:

```
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere
Nginx Full                 ALLOW       Anywhere
OpenSSH (v6)               ALLOW       Anywhere (v6)
Nginx Full (v6)            ALLOW       Anywhere (v6)
```

You can now test the server by going to `http://YOUR_IP_ADDRESS`. You should see "Welcome to nginx!" message.

Open the file:

```bash
sudo nano /etc/nginx/nginx.conf
```

Find the `server_names_hash_bucket_size` directive and remove the `#` symbol to uncomment the line. Also add `client_max_body_size` with your desired request size limit.

```
...
http {
    ...
    client_max_body_size 64M;
    server_names_hash_bucket_size 128;
    ...
}
...
```

Save and close when you are finished. Doing this allows you to use longer domain names.

## 2. Install MySQL

Run the following command:

```bash
sudo apt-get install mysql-server
sudo mysql_secure_installation
```

1. When asked to enable `VALIDATE PASSWORD COMPONENT` select no.
2. After that, enter the DB password that you want to use.
3. Now it'll ask you 4 questions about removing dummy data. Enter 'Yes' for all of them.

Now you can access MySQL using:

```bash
sudo mysql
```

Notice that you didn’t need to provide a password to connect as the root user, even though you have defined one. That is because the default authentication method for the administrative MySQL user is `unix_socket instead` of `password`. Even though this might look like a security concern at first, it makes the database server more secure because the only users allowed to log in as the root MySQL user are the system users with sudo privileges connecting from the console or through an application running with the same privileges.

It’s best to have dedicated user accounts with less expansive privileges set up for every database

### Adding More Users

To add another user to your database, you can run this query in MySQL prompt:

```sql
CREATE USER 'MY_USER_NAME'@'localhost' IDENTIFIED BY 'MY_PASSWORD';
```

To give the user permissions for a particular database:

```sql
GRANT ALL PRIVILEGES ON MY_DATABASE_NAME.* TO 'MY_USER_NAME'@'localhost' WITH GRANT OPTION;
```

## 3. Install PHP

Run this command to install PHP, some PHP extensions and composer. You can add/remove extensions as per your requirements:

```bash
sudo apt-get install php-fpm php-mysql php-cli php-curl php-gd php-intl php-mbstring php-soap php-xml php-xmlrpc php-zip composer

```

Now we need to make a slight configuration change:

```bash
sudo nano /etc/php/7.X/fpm/php.ini # Replace X with your version of PHP
```

In this file, look for `cgi.fix_pathinfo`. It should be commented out using `;`
Remove the comment and set the value to 0: `cgi.fix_pathinfo=0`

Apart from this, there are some more useful properties which you can configure according to your requirement:

1. `memory_limit = 128M`
2. `max_execution_time = 30`
3. `post_max_size = 8M`
4. `upload_max_filesize = 2M`
5. `max_file_uploads = 20`

Once done, restart PHP processor by typing:

```bash
sudo systemctl restart php7.X-fpm # Replace X with your version of PHP
```

## 4. Setting up Nginx Server Blocks

By default Nginx will server files from `/var/www/html` which is fine if you are only serving a single domain. If you want to use multiple domains, you'll need to setup server block for each domain.

First start by making a folder with the right ownership and permissions:

```bash
sudo mkdir /var/www/example.com
sudo chown -R www-data:www-data /var/www/example.com
sudo chmod -R 755 /var/www/example.com
```

Next, we'll make a server block by typing this command:

```bash
sudo nano /etc/nginx/sites-available/example.com
```

In case of a simple HTML website enter this and save:

```
server {
        listen 80;
        listen [::]:80;

        root /var/www/example.com;
        index index.html index.htm;

        server_name example.com www.example.com;

        location / {
                try_files $uri $uri/ =404;
        }
	
        location ~ /\.ht {
                deny all;
        }
}
```

In case of a Vue/React/Angular or SPA app with a router enter this and save:

```
server {
        listen 80;
        listen [::]:80;

        root /var/www/example.com;
        index index.html index.htm;

        server_name example.com www.example.com;

        location / {
                try_files $uri $uri/ /index.html;
        }
	
        location ~ /\.ht {
                deny all;
        }
}
```

In case of a simple PHP application enter this and save:

```
server {
        listen 80;
        listen [::]:80;

        root /var/www/example.com;
        index index.php index.html index.htm;

        server_name example.com www.example.com;

        location / {
                try_files $uri $uri/ =404;
        }
	
        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php7.X-fpm.sock;
        }
	
        location ~ /\.ht {
                deny all;
        }
}
```

In case of a Laravel application, enter this and save:

```
server {
    listen 80;
    listen [::]:80;

    root /var/www/example.com/public;
    index index.php index.html index.htm;

    server_name example.com www.example.com;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php7.X-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

In case of a WordPress website, enter this and save:

```
server {
    listen 80;
    listen [::]:80;

    root /var/www/example.com;
    index index.php index.html index.htm;

    server_name example.com www.example.com;

    location / {
        try_files $uri $uri/ /index.php$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php7.X-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

**Note:** Make sure to replace the 7.X in in all the above with your version of PHP.

Now to enable this file, run this command:

```bash
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
```

Repeat this process for each domain that you want to add.

Finally test your nginx configuration and reload:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Once you have created all the blocks you need, you can disable the default block using:

```bash
sudo unlink /etc/nginx/sites-enabled/default
```

## 5. Install & Secure phpMyAdmin

Download and install using this command:

```bash
sudo apt-get install phpmyadmin
```

1. During the installation you'll be asked which webserver should be auto configured. Since nginx is not in that list, just press `TAB` and `ENTER` to skip.
2. The next prompt will ask if you would like `dbconfig-common` to configure a database for phpMyAdmin to use. Select “Yes” to continue. You’ll need to enter the database administrator password entered in step 2 above.

After installation, the executable files can be found in `/usr/share/phpmyadmin/`. To make this accessible via the browser you have to make a symlink between this folder and any publicly accessible folder in `/var/www/`. For example:

```bash
# In case you want it accessible by http://YOUR_IP_ADDRESS/phpmyadmin
sudo ln -s /usr/share/phpmyadmin /var/www/html

# In case you want it accessible by http://phpmyadmin.example.com
sudo ln -s /usr/share/phpmyadmin /var/www
sudo mv phpmyadmin phpmyadmin.example.com
```

**Note:** For using it in `http://phpmyadmin.example.com`, you will have to create another server block by following the instructions in step 4 above.

### Securing phpMyAdmin with a Password

To create an encrypted password type:

```bash
openssl passwd 'YOUR_PASSWORD_HERE'
```

**Note:** Your password cannot be longer than 8 characters.

After you run this command, you will get an encrypted version of the password like this:

```
hbfHQTlXYYSOo
```

Copy this value as it'll have to be pasted somewhere in a bit. Now, create an authentication file by typing

``` bash
sudo nano /etc/nginx/pma_pass
```

In this file, you’ll specify the username you would like to use, followed by a colon (`:`), followed by the encrypted version of the password you received from the `openssl passwd` utility. For example:

```
myusername:hbfHQTlXYYSOo
```

Save and close the file when you’re done. Now, we’re ready to modify our Nginx configuration file. Open it in your text editor to get started:

```bash
# In case you're running it on http://YOUR_IP_ADDRESS/phpmyadmin
sudo nano /etc/nginx/sites-available/default

# In case you're running it on http://phpmyadmin.example.com
sudo nano /etc/nginx/sites-available/phpmyadmin.example.com
```

In this configuration, you need to specify the the authentication file.

In case you're running it on `http://YOUR_IP_ADDRESS/phpmyadmin`, use this configuration:

```
server {
    . . .
    
    location / {
        try_files $uri $uri/ =404;
    }

	location /phpmyadmin {
        auth_basic "Admin Login";
        auth_basic_user_file /etc/nginx/pma_pass;
    }

    . . .
}
```

In case you're running it on `http://phpmyadmin.example.com`, use this:

```
server {
    . . .
    
    location / {
        try_files $uri $uri/ =404;
        auth_basic "Admin Login";
        auth_basic_user_file /etc/nginx/pma_pass;
    }

    . . .
}
```

Save and close the file when you’re done. To activate our new authentication gate, we must restart the web server:

```bash
sudo service nginx restart
```

## 6. Enable HTTPs with Let's Encrypt

Install Certbot and it’s Nginx plugin with `apt`:

```bash
sudo apt install certbot python3-certbot-nginx
```

Obtain an SSL certificate by running the following command after replacing example.com with your domain:

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

If that’s successful,  `certbot`  will ask how you’d like to configure your HTTPS settings.

```
Output
Please choose whether or not to redirect HTTP traffic to HTTPS, removing HTTP access.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: No redirect - Make no further changes to the webserver configuration.
2: Redirect - Make all requests redirect to secure HTTPS access. Choose this for
new sites, or if you're confident your site works on HTTPS. You can undo this
change by editing your web server's configuration.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate number [1-2] then [enter] (press 'c' to cancel):
```

Select your choice and hit `ENTER`.  And done !
