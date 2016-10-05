# CSD-2323 Assignment 4

This set of files includes steps to further configure and secure WordPress on a
CentOS 7 LAMP Server.

For other systems (eg- Ubuntu, Debian, Arch, etc...) refer to the specific
distribution's documentation for default file locations and general steps.

NOTE: Reminder that you may need to run `sudo setenforce 0` to turn off SELinux
for the early part of the exercise. We'll fix it later, don't worry.



## HTTPS Certificates

To gain the S part of HTTPS, you need to install an SSL/TLS certificate. Ideally
you would be installing this on a live web server using a CA like [Let's Encrypt](https://www.letsencrypt.org)
but for this exercise we will be installing a self-signed certificate, and then
trusting that in the browser.

Our general steps will be:

* Install `mod_ssl`
* Create a Self-Signed Keypair
* Update the Browser to Trust the Certificate


### Install `mod_ssl`

Apache is a modular web server. One of its most important modules is `mod_ssl`,
which provides the SSL/TLS integration for the Apache Web Server.

To install this into CentOS use:

```bash
sudo yum install mod_ssl
```


### Creating a Self-Signed Keypair

Our next step is creating an SSL/TLS keypair that Apache can use to encrypt its
incoming and outgoing messages. We are going to store these in a central location
and create a new self-signed keypair.

```bash
sudo mkdir /etc/httpd/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/httpd/ssl/blog.key -out /etc/httpd/ssl/blog.crt
```

The second command, in brief: creates an X.509 certificate with an unencrypted
private key that expires in 365 days. The keys use RSA with 2048bit key size.
The private key is sent to `/etc/httpd/ssl/blog.key` while its public key (cert)
is sent to `/etc/httpd/ssl/blog.crt`.

During this process, you will be asked several questions. In a production
environment, with a live server, the answers to these questions could mean the
difference between your certificate being validated or not. Here, however, the
most important thing is the **Common Name**. This is the hostname of your server.

When you are asked for **Common Name** provide it with `blog.server.local`. The
other answers can simply be default.


### Changing Apache's Config to Use SSL

At this point, we want to change the previous VirtualHost configuration to use
SSL/TLS instead of unencrypted transfers.

```bash
sudo nano /etc/httpd/conf.d/virtual.conf
```

You can remove the previous `<VirtualHost *:80>` block for blog.server.local.

Then add in the following:

```aconf
<VirtualHost *:443>
    DocumentRoot            /home/blog/public_html
    ServerName              blog.server.local:443
    SSLCertificateFile      /etc/httpd/ssl/blog.crt
    SSLCertificateKeyFile   /etc/httpd/ssl/blog.key
</VirtualHost>
```

With this in place, you are able to restart Apache:

```bash
sudo systemctl restart httpd
```

When you navigate to [https://blog.server.local/](https://blog.server.local/) in
a browser, though, your browser will almost certainly give you an Untrusted
Connection error. This is because our certificate is self-signed. So at this
point the connection is encrypted, but not trusted.


### Make the Browser Trust the Certificate

Because there are so many different possibilities for which browser and which
base OS you're using, I won't go into specific details of this process for all
possibilities, but I will outline the steps for Firefox in CentOS 7.

When you navigate to the page, you should see something that looks like this:

![SSL Error Page 1](https://github.com/LenPayne/CSD2323-Assign4/raw/master/ssl-error-01.png)

Click "Advanced" and then "Add Exception..."

![SSL Error Page 2](https://github.com/LenPayne/CSD2323-Assign4/raw/master/ssl-error-02.png)

Read all the warnings about why this is generally a bad idea. Then select
"Confirm Security Exception" anyway, because this is what we want in this case.

![SSL Error Page 3](https://github.com/LenPayne/CSD2323-Assign4/raw/master/ssl-error-03.png)

At this point, you should be able to navigate to [https://blog.server.local/](https://blog.server.local/)
and see a "green lock" that indicates the site is trusted and secure.



## Rehashing Permissions for Best Practices

Permissions are an important step of locking down a system. Securing transmissions
are important, but making sure that an attacker is locked down is equally important.
To address this, we'll look at two case studies.

### Our Static Site `www.server.local`

As of right now, [http://www.server.local/](http://www.server.local/) is a
functional and working static website. Static means it won't change. Since it
won't change, nothing should have write access on it.

Use the following commands to ensure the files are not writable by anyone but
`root`:

```bash
sudo chown root.root -R /var/www/html
sudo chmod -R 644 /var/www/html
```

### Our Blog/Dynamic Site `blog.server.local`

However, our [https://blog.server.local/](https://blog.server.local/) site is
very much dynamic, and needs more fine-grained permissions.

Since we're using WordPress, it's important to refer to the [WordPress Hardening](https://codex.wordpress.org/Hardening_WordPress) article from the WP
Codex.

Basically, we want to run the following commands to set us to a bare-bones:

```bash
sudo chown -R blog.apache /home/blog/public_html
sudo find /home/blog/public_html -type d -exec chmod 755 {} \;
sudo find /home/blog/public_html -type f -exec chmod 644 {} \;
```

These commands effectively:

1. Ensure that the blog files are all owned by `blog` and in the `apache` group.
2. Finds all the directories in the WordPress folder and gives them 755 perms.
3. Finds all the files in the WordPress folder and gives them 644 perms.

We also need to make a couple of exceptions to allow for WordPress to manage
URL Rewrites (useful for making nice URLs), and allowing uploads.

```bash
sudo chmod g+w /home/blog/public_html/.htaccess
sudo mkdir /home/blog/public_html/wp-content/uploads
sudo chown blog.apache /home/blog/public_html/wp-content/uploads
sudo chmod -R g+w /home/blog/public_html/wp-content/uploads
```

At this point, you should be able to upload files to WordPress. Give it a shot.



## Managing SELinux Correctly

Previously, we turned off SELinux.

To see proof of this, run the following:

```bash
getenforce
```

It should say `Permissive`. There are three possible statuses for enforcement:
`Permissive` is open, but warns when rules would be flagged, `Enforcing` enforces
the rules by blocking unwanted behaviour, and `Disabled` is wide open with no
blocks or warnings.

We're going to turn it back on. At the end of the following, it should be
`Enforcing`:

```bash
sudo setenforce 1
getenforce
```

If you try to connect to your server, you'll see that [http://www.server.local](http://www.server.local)
works just fine, but [https://blog.server.local](https://blog.server.local)
throws a 500 error. Apache isn't allowed access to the folders in `/home/blog`
any more because we turned on the Mandatory Access Control rules of SELinux.

To read the context information, run the following:

```bash
ls -Z /home/blog
```

You will see something like:

```
drwxr-xr-x. blog apache unconfined_u:object_r:httpd_user_content_t:s0 public_html
```

This listing includes the SELinux context information. Most notably, it tells us
that `public_html` is considered `httpd_user_content_t`, which is a Type that
has special rules on it.

To manage those rules, we need to install a set of Python-based SELinux tools:

```bash
sudo yum install policycoreutils-python
```

This allows us to run the following command, which will let us see all the on/off
rules set up in SELinux that have to do with `httpd`:

```bash
sudo semanage boolean -l | grep httpd
```

One of these rules is `httpd_read_user_content`, which allows `httpd` to read
things that are of the `httpd_user_content_t` type.

We turn that rule ON with:

```bash
sudo setsebool -P httpd_read_user_content on
```

With that rule changed, we should now be able to go to [https://blog.server.local](https://blog.server.local)
safely, with SELinux still turned on. As a bonus, all other UserDir folders will
also work correctly.



## Installing wp-cli and Wordfence

The site is fairly secure, now all we need to do is lock down WordPress. The big
first step is to install `wp-cli` which is a great tool that allows you to manage
a lot of WordPress functionality from the command line. This means if we want to,
we can turn off a lot of the write functionality on the WordPress folders.

To install `wp-cli` we can follow the instructions on [http://wp-cli.org](http://wp-cli.org)
or just follow below:

```bash
wget https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
chmod +x wp-cli.phar
sudo mv wp-cli.phar /usr/local/bin/wp
```

If the above worked correctly (which it should, since you definitely have PHP
installed and configured,) then you should be able to run `wp --info` and get
some meaningful feedback.

You can run `wp` as any user, but it's best to actually do this part as `blog` to
keep the permissions correct. So we'll use `su` to log in as `blog`. We're using
`su` instead of `sudo su` because `su` retains all the proper login functionality
correctly: most importantly it keeps `/usr/local/bin` in the user's `$PATH`.

So anyway, log in as `blog` and play around with `wp`:

```bash
su blog
cd ~/public_html
wp plugin update-all
wp theme update-all
wp theme install sydney
wp theme activate sydney
```

At this point, you should be able to navigate to your blog and notice that you
have a completely new look. To scan for more themes, head over to [https://wordpress.org/themes](https://wordpress.org/themes).
You can install by package name (eg- `wp theme install olsen-light`) or by plain
readable name (eg- `wp theme install "Olsen Light"`).


### Installing Wordfence

Wordfence is one of the best comprehensive security plugins available for WordPress.
Install it and activate it.

```bash
wp plugin install wordfence --activate
```

When you navigate to the WordPress Dashboard [https://blog.server.local/wp-admin](https://blog.server.local/wp-admin)
you will see a new Wordfence icon down on the left. You will also be prompted to
setup some basic configuration for Wordfence. Follow through the tour and
tutorial and set up your security.

At this point, your site is fairly secure. Great job.


### Finishing Touch - HTTP Redirect

In the current state we have it, you may be seeing your site functioning at [https://blog.server.local/](https://blog.server.local/)
but not at [http://blog.server.local/](http://blog.server.local/). The best
practice we want to do is set up a 301 Redirect from the http path to https.

We do this in `virtual.conf`.

```bash
sudo nano /etc/httpd/conf.d/virtual.conf
```

Add the following configuration to redirect all http:// requests on blog.server.local
to https:// requests.

```aconf
<VirtualHost *:80>
    ServerName blog.server.local
    Redirect / https://blog.server/local
</VirtualHost>
```


## Hand it In

So navigate to: [https://blog.server.local/wp-admin](https://blog.server.local/wp-admin)
and take a screenshot of the WordPress Dashboard with the green lock for HTTPS,
and the Wordfence icon visible as installed. Submit the screenshot of it to D2L.

![Sample Screenshot](https://github.com/LenPayne/CSD2323-Assign4/raw/master/finished-screencap.png)

Note: You may not have exactly the same results, as I took my screenshot after
installing some plugins and themes, but the most important part is having the
WordPress Dashboard up and running.

## Optional Challenges for You to Explore

1. Set up `vsftpd` or another FTP daemon to enable Plugin and Theme installs from
   the WordPress Dashboard.
2. Install a Theme and setup your website with Posts and Pages. Have fun!
3. Install some other security plugins like `cloudflare` and `sf-move-login` and
   `disable-xml-rpc-pingback`
