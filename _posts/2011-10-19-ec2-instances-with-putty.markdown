
I use a windows machine at work, so I use Putty to access all of my fun Linux goodies up in the cloud.  I have several EC2 EBS based instances that I'm continually stopping and starting for various reasons (mostly to save money).
However, my Putty workflow was a royal pain. Every restart of an instance required waiting for it to boot, finding out its new public address, and then load that address into Putty to connect. After I strained my pinky after too much ctrl-C/ctrl-V, I decided to find a way to automate the process.  Google to the rescue!! or so I thought. I couldn't find anything that solved my problem out of the box on Google, probably due to my (lack of good) Google-fu. I was left to my own devices, and by finding little hints and clues from various places I finally created a simple one-click login work flow.

### Do the two step
It seemed pretty simple, I just need to do a couple of things.

* Get the updated public address for the restarted instance
* Somehow get that address as a parameter to Putty

###Who am I
Well for this to work at all I needed a way to get the instance address.  What I wanted was to be able to use a meaningful name to me (Like: MyServer) as the handle and have all the details go on behind the scenes.

_RatHole #1: ec2-api tools. I won't bore you with my floundering, but I didn't find an elegant way to do what I wanted.  Not only that, I wasn't really keen on having an ec2-api dependence. I didn't use them in my normal, cumbersome workflow, why include them now if I didn't have to. It didn't help that it's heavily dependent upon the local environment._

Luckily a [Stack Overflow](http://stackoverflow.com/questions/7525702/how-to-find-the-ip-address-of-an-amazon-ec2-instance-on-reboot) question provided an alternate way. From within an instance you can use:
{% highlight bash %}
    http://instance-data/latest/meta-data/public-ipv4
{% endhighlight %}
to query the instance metadata for its public IP. A quick little
{% highlight bash %}
    curl -s http://instance-data/latest/meta-data/public-ipv4
    #=> "174.129.1.1"
{% endhighlight %}
confirmed it.

### Ok, but who am I now?
  Great, but how to get the new data when I restart the instance?  Fortunately, that wasn't as hard as I feared it would be. The special cron command `@reboot` is triggered when an instance is restarted.  So now I have a hook to call the script when the instance is restarted.  Also, I learned that if you're using rvm, that you have to be careful with cron jobs, but a quick little:
{% highlight bash %}
     crontab -e
{% endhighlight %}
to open the file and inserting
{% highlight bash %}
    @reboot /bin/bash -l -c 'ruby /home/me/whoami_test.rb'
{% endhighlight %}
into the crontab file did the job.

### Telling someone who I am
Wonderful, I can get the instance to tell itself who it is on a reboot.  How does that help me in Windowsland unconnected to the instance because I don't have a reference to it?  Somehow I have to export the address to a place where Windows would have access to it.

_Rathole #2: Perfect fit for S3 or its close cousin Simple Database I thought.  Except that a simple http request isn't going to get you anywhere, AWS required encryption of the keys. This wasn't a huge problem as there were AWS libraries to do this, and I had myself updated an older Aws-Sdb library to update from an outdated encryption algorithm. I could have just included the libraries on both the server and windows machines and used those ruby libraries, but I felt that there had to be a cleaner way._

_Rathole #3: Create an API on one of my webservers.  This wasn't so much a rathole as I didn't want to create something new when there surely had to be something out there already._

I ended up going with CouchDB, specifically [Iris Couch](http://www.iriscouch.com/)  Which lets you quickly and easily set up a database.  Also, I'm quite familiar with CouchDB having used it for a few other projects, and have installed CouchDb on my own servers a few times.  Anyway, one of the great things about CouchDB is that you use standard REST commands (GET,PUT,POST,DELETE) and any user data is plain json.  So using Iris and CouchDB's Futon admin utility I created a database, let's call it `"bootie"`, and populated it with `{"_id":"MyServer", "ip":"not relevant yet"}`. Although I did this manually, it's quite feasible to do programmatically, I was just being lazy since it's a one time setup. 

To actually update a CouchDB record, requires a two step process. One to get the current revision of the record, and the second to put the updated data, including the current revision to be updated. It sounds complicated, but its really not that hard in practice. So now, my server side program looked like this.
{% highlight ruby %}
    require  'json'
    #Get Instance IP Address
    my_ip = `curl -s http://instance-data/latest/meta-data/public-ipv4`

    my_server_name = "MyServer"
    bootstrap_db = "http://my_iris.iriscouch.com/bootie"

    #url for the record data
    record_loc = bootstrap_db + "/" + my_server_name

    #get rev of current record
    data_raw = `curl -s GET #{record_loc}`
    data = JSON.parse(data_raw)
    rev = data["_rev"]

    #update database to current value
    new_record = {
      "_id" => my_server_name,
      "_rev" => rev,
      "ip" => my_ip
    }
    new_record_json = new_record.to_json
    resp = `curl -s --request PUT #{record_loc} -Hcontent-type:application/json -d \'#{new_record_json}\'`
{% endhighlight %}
Some quick notes: I used the native curl to interface with the database, I could have any one of several ruby libraries, including Net::Http, but I chose curl. Also, this approach may not be for you if security is paramount. It's secure enough for my needs, but I don't have any production or sensitive data running on any of these servers. Lastly, this script isn't bullet-proofed at all. It's not hard to do the error and validation checks, just time-consuming and this already does enough of what I need it to do for my purposes.

A quick test by restarting the instance had me seeing the updated IP address in Iris.  Half way there!

### Telling Putty who I am

Ok, now I just need to get the IP to Putty and things should be golden.

_Rathole #4: Since I was on Windows I thought I'd stay closer to Windows programming by using Vbscript. It turns out that there was no native support for parsing JSON, and to do so would require jumping through more hoops than I desired._

I ended up going with Ruby as the scripting language on the windows machine too.  First, we need to get the server IP from the Iris. Since I had gone this far doing a fairly decent job of avoiding external libraries, I decided to see if `open-uri` from the Ruby Standard Library would do what I needed it too.
{% highlight ruby %}
    require 'open-uri'

    Server = "MyServer"
    bootstrap_data_json = open('http://forforf.iriscouch.com/bootstrap/' + Server){|f| f.read}
{% endhighlight %}
Success!! Iris responded without errors

_Rathole #5: I initially went with the awesome `rest-client` gem, and that worked fine, but it ended up being an unnecessary external dependency._

Getting the handle for the server name, and it's IP address was straightforward:
{% highlight ruby %}
    require 'open-uri'

    Server = "MyServer"
    bootstrap_data_json = open('http://forforf.iriscouch.com/bootstrap/' + Server){|f| f.read}

    bootstrap_data_raw = JSON.parse(bootstrap_data_json)
    srv_name = bootstrap_data_raw["_id"]
    ip_addr = bootstrap_data_raw["ip"]
{% endhighlight %}
Now, all I had to do was kick off Putty with this data.

_Rathole #6: Formatting directory paths in Windows to be compatible with Ruby and native system calls._

Fortunately, Putty has a robust command line, and using the `-i` command to include the authentication credentials allowed me to clear the biggest hurdle.  The command ended up looking like this:
{% highlight ruby %}
    putty_cmd = "start putty.exe" \
                " -i \"C:\\Documents and Settings\\path\\to\\my\\keypair.ppk\" \
                " ec2-user@#{ip_addr}"
{% endhighlight %}
The complete script for windows ended up looking like this:
{% highlight ruby %}
    #ssh_my_server.rb

    require 'open-uri'

    Server = "MyServer"
    bootstrap_data_json = open('http://forforf.iriscouch.com/bootstrap/' + Server){|f| f.read}

    bootstrap_data_raw = JSON.parse(bootstrap_data_json)
    srv_name = bootstrap_data_raw["_id"]
    ip_addr = bootstrap_data_raw["ip"]

    putty_cmd = "start putty.exe" \
                " -i \"C:\\Documents and Settings\\path\\to\\my\\keypair.ppk\" \
                " ec2-user@#{ip_addr}"
{% endhighlight %}

Opening a windows command window and typing `ruby ssh_my_server.rb` opens up a Putty terminal connected to my instance. Sucesss!! No more hunting down address changes, and my pinky will have time to heal now.

PS: I plan on wrapping the the ruby script in a bat file or vbscript so that I can click on the script in file explorer. I haven't done it yet, but a batch file like this:
{% highlight bash %}
    ruby /path/to/script/ssh_my_server.rb
{% endhighlight %}
would do the trick, I imagine.

##TL;DR
### Setup a CouchDB

   [Iris](http://www.iriscouch.com/)

###Server Script
{% highlight ruby %}
    require  'json'
    #Get Instance IP Address
    my_ip = `curl -s http://instance-data/latest/meta-data/public-ipv4`

    my_server_name = "MyServer"
    bootstrap_db = "http://my_iris.iriscouch.com/bootie"

    #url for the record data
    record_loc = bootstrap_db + "/" + my_server_name

    #get rev of current record
    data_raw = `curl -s GET #{record_loc}`
    data = JSON.parse(data_raw)
    rev = data["_rev"]

    #update database to current value
    new_record = {
      "_id" => my_server_name,
      "_rev" => rev,
      "ip" => my_ip
    }
    new_record_json = new_record.to_json
    resp = `curl -s --request PUT #{record_loc} -Hcontent-type:application/json -d \'#{new_record_json}\'`
{% endhighlight %}  
###Client Script
{% highlight ruby %}
    #ssh_my_server.rb

    require 'open-uri'

    Server = "MyServer"
    bootstrap_data_json = open('http://forforf.iriscouch.com/bootstrap/' + Server){|f| f.read}

    bootstrap_data_raw = JSON.parse(bootstrap_data_json)
    srv_name = bootstrap_data_raw["_id"]
    ip_addr = bootstrap_data_raw["ip"]

    putty_cmd = "start putty.exe" \
                " -i \"C:\\Documents and Settings\\path\\to\\my\\keypair.ppk\" \
                " ec2-user@#{ip_addr}"
{% endhighlight %}
###Run It
`ruby ssh_my_server.rb` or wrap it in a batch file for clickable goodness
