[![Build Status](https://travis-ci.org/PhilWaldmann/squid3_sentry.png)](https://travis-ci.org/PhilWaldmann/squid3_sentry)

# squid3_sentry

a redirect_program and icap Server for squid3 (like squidGuard, but more flexible)

## Highlights
* Pure Javascript
* Easily extendable
* Rich set of predefined rule templates
* LDAP compatible
* Dynamic rule injection (no reload necessary)

## Dependencies
* [Node.js](http://nodejs.org/)
* [Squid3](http://www.squid-cache.org/)
* Optional: LDAP Server (squid with user auth. e.g. ntlm)
* Optional: Redis Server (dynamic rules and large domains lists e.g. [shallalist](http://www.shallalist.de/))

## Install sentry
Install Sentry globally:

    npm install squid3_sentry -g

Or install it locally:

    npm install squid3_sentry

## Configuration (sentry)
### via params
For a list of all params run

    $ sentry --help
  
### via config file
Config files need to be written in json (See examples folder)

Params:

* ```name```: The name of your instance. This is optional if you just have one squid instance!
* ```redirect```: The url to redirect the user if something is blocked
* ```mode```: Global mode for ```redirect``` or ```rewrite```. See ```mode``` in rules definition.
* ```log```: Path to the log file. [Bunyan](https://github.com/trentm/node-bunyan) is used as a logger.
* ```log_level```: Use ```error```, ```warn``` or ```info```. Default is ```error```.
* ```cache_time```: Cache time in milliseconds. Cache will be cleated after that time (e.g. 300000 for 5 mins)
* ```timeout```: The timeout per request in milliseconds.
* ```range```: This will used for the balancer only (See Balancer)
* ```ldap url```: The url to your ldap server (e.g. ```ldap://domain.local```)
* ```ldap dn```: The path to the user which will query your ldap directory (e.g. ```CN=MyUser,CN=Users,DC=domain,DC=local```)
* ```ldap password```: The password for that user
* ```ldap base```: The base path for all searches
* ```redis host```: url or domain of your redis host (e.g. ```localhost```)
* ```redis port```: The port of your redis server
* ```redis password```: If you use redis authentication.
* ```rule_sources```: Array of sources. Currently only 'config' and 'redis' is available. The rules are positioned depending on the source position in the array
* ```rules```: Array of rule definitions. (only via config file)

for debugging:

* ```measure_times```: The amount of time for every request will be  messured. The times are visible in the debugger (See Live Debugging)
* ```dry_run```: Runs sentry in test mode. Every request is allowed, but sentry still checks it's rules! (See Live Debugging)

## Configuration (squid3)
Add the following to your ```squid.conf```

    redirect_program sentry your/config/file.json
    redirect_children 1
    redirect_concurrency 100

## Rules
The first rule that matches (e.g. all given criteria matches) will be executed (Deny or Allow).

The following rule will deny all requests and squid will redirect all http connections to the globally defined ```redirect``` address:

    {
      name: 'rule1',
      allowed: false
    }
    
To define a custom redirect address:

    {
      name: 'rule1',
      allowed: false,
      redirect: 'my.custom.redirec.com'
    }

The following configuration options are available:

* ```name```: Name of the rule (For debugging/verbode mode only). Default: rule<nr>
* ```allowed```: Deny or allow the url (e.g. black- or whitelist). Default: false
* ```mode```: Instead of redirecting a browser to another url (302 HTTP header) you could set it to ```rewrite``` to tell squid to load another page instead of redirecting the browser
* ```redirect```: The redirect or rewrite url

### Redirect config
You could use the following placeholders:

* ```[url]```: Current url
* ```[domain]```: Current Domain
* ```[name]```: Matching rule name
* ```[ip]```: IP of the incomming connection
* ```[username]```: Username of the incomming connection (if squid uses authentication)
* ```[method]```: HTTP method. e.g. GET, POST,..

### Default criteria
You could configure your rules with any of the following criteria:

* ```categories```: Array of category names. A category is a list of domains and/or urls stored in redis. (See shallalist import)
* ```category_files```: Same as categories, but expects a file paths with domains and/or urls. These files will be stored in redis and watched for changes
* ```file_types```: Array of filet ypes (e.g. ```['swf', 'flv']```)
* ```ips```: Array of ips (e.g. ```['10.20.30.0/24', '10.55.11.33']```) 
* ```matches```: Array of wildcard matches (e.g. ```['*goog*']```)
* ```groups```: Array of LDAP groups. The full LDAP path is expected (e.g. ```['CN=MyGroup,CN=Users,DC=domain,DC=local']```) 
* ```users```: Array of usernames
* ```ous```: Array of LDAP OUs. The full LDAP path is expected (e.g. ```['OU=Development,DC=domain,DC=local']```)
* ```times```: Array of objects in the following format: {from: '08:00', to:'17:00', week_day: 1}.
  The following params are available
 * from: From time (24h)
 * to: To time (24h)
 * week_day: JS Date week date
 * day
 * month
 * year

### Custom rule definitions
To extend sentry and add custom rules take a look at ```lib/rules/```

Example rule which will deny every second request
    
    Squid.start(config);
    Squid.core.addRuleDefinition({
      name: 'my_rule_def',

      //will be called once for every rule - in the context of the rule.
      config: function(options){
        //This rule definition is active (filter will be called!)
        this.types.push('my_rule_def');
        this.allow_next = true;
      },

      //will be called for every request for every rule where my_rule_def is active
      filter: function(options, callback){
        //Return true if the rule definition matches.
        callback(null, this.allow_next);
        //Alternate every request
        this.allow_next = !this.allow_next;
      }
    });
    
    //Add a rule if there isn't any defined
    Squid.core.addRule({
      allowed: false
    });

## Shallalist import
Use the ```import``` script to import shallalist into redis

    $ import path/to/shallalist


## Live Debugging
Use the ```debugger``` script for life debugging

    $ debugger config.json
    
## Balancer
Use the ```balancer``` script for round robin style balancing.
The following command will start a balancer with 3 sentry processes:

    $ balancer config.json config.json config.json
    
You can also use the ```balancer``` script to create a separate context for different subnets.
E.g. 10.20.30.0/24 should have a different set of rules than 192.168.0.0/24.
Create a config file for every subnet and put the subnet definition into the ```range``` config option and start the blancer with the following command:

    $ balancer subnet1.json subnet2.json
