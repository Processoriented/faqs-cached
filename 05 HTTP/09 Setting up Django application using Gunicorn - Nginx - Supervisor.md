# Setting up Django application using Gunicorn - Nginx - Supervisor

        
There have been several very good detailed tutorials written on the how to set up a Django application(s) using Gunicorn as the application server, Nginx as the proxy server/static content web server, and Supervisor to monitor and manage it.

So I'll not spend much time rehashing what those tutorials tell you, but sharing the details of what it took to get it up and running here:

Most of the tutorials refer to PostgreSQL setups, but MySQL is already available with your account and supported by Django, and there is sqlite which is included in the base python installation. I chose the later as my database already existed in sqlite.

The biggest challenge was on the nginx side. All the tutorials reference to setting up a "server"/virtual host and making changes within /etc/nginx/sites-available and /etc/nginx/sites-enabled. Those are not accessible for change within a slot. Therefore leverageing location {} was the only option that would work (3+ days of trial and error to find it).

I created two files in:

```shell
~/.nginx/conf.d/000-default-server.d
```

### static-app.conf
Create a file called:

```shell
~/.nginx/conf.d/000-default-server.d/static-app.conf
```

And populate it with this. Edit in your info:

```nginx
location /static/ {
alias /media/DiskID/home/username/www/$host/public_html/webapps/static/;
}
```

### <app-name>-app.conf

Create a file called, where *YourApp* is the name of the application:

```shell
~/.nginx/conf.d/000-default-server.d/YourApp-app.conf
```

And populate it with this. Edit in your info:

```nginx
location /<app-name>/ {

# an HTTP header important enough to have its own [`Wikipedia entry`](http://en.wikipedia.org/wiki/X-Forwarded-For):
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

# pass the Host: header from the client right along so redirects
# can be set properly within the Rack application
proxy_set_header Host $http_host;

# we don't want nginx trying to do something clever with
# redirects, we set the Host: header above already.
proxy_redirect off;

proxy_pass http://<internal-ip-address:port>;

}
```

That second file was the key; When you go to your browser and type in:

`http://username.server.feralhosting.com/app-name/`

It will proxy/redirect the request (including all the parts of the URI from */<app-name>/....* to end).

The address of *http://Internal-IP-address:port* is the same bind/address that was set-up within Gunicorn.

To get internal IP address, I was able to use the following command:

```shell
/sbin/ifconfig |grep -B1 "inet addr" |awk '{ if ( $1 == "inet" ) { print $2 } else if ( $2 == "Link" ) { printf "%s:" ,$1 } }' |awk -F: '{ print $1 ": " $3 }'
```

The address next to users: is the IP address you want to leverage. **(I'm assuming it does stay static)**

The purpose of that is to avoid using a public facing IP address like the one associated with <slot>.feralhosting.com which could cause you to expose the application directly to the outside world which can be a security problem for you.

The last piece of the puzzles that is different from all other set-ups is within the Django area. If you leverage the admin capability of Django, you will need to update the default set-up for *urls.py* related to admin.

```python
# Uncomment the next line to enable the admin:
# url(r'^admin/', include(admin.site.urls)),
url(r'^<app-name>/admin/', include(admin.site.urls)),
```

This way Django will be expecting a URL starting with *app-name/admin/* instead of *admin/*. This will help you with regards to supporting multiple applications and using the different Admin pages for each application separately. 

Everything else basically came from the tutorial provided below, and there are others on the web that provide slight variations of it. But the above items should help with getting things set-up on your slot. 

My next step is to work on creating a script that will handle automating alot of these steps to make sure it gets done correctly each time without forgetting a step. I'll share a link to that once completed in case others would like to leverage it.

Good luck!!!

### References
[Django](https://www.djangoproject.com/)
[Gunicorn](http://gunicorn.org/)
[Nginx](http://nginx.org/)
[Supervisor](http://supervisord.org/)
[Virtualenv](http://www.virtualenv.org/en/latest/)
[Setup Tutorial Reference](http://michal.karzynski.pl/blog/2013/06/09/django-nginx-gunicorn-virtualenv-supervisor/)

