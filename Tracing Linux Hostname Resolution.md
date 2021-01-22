# **Tracing Linux Hostname Resolution**

Let’s examine hostname resolution on a RHEL 5.5 box on a Sunday night. I was inspired from reading [Down the ‘ls’ Rabbit Hole](http://sysadvent.blogspot.com/2010/12/day-15-down-ls-rabbit-hole.html) 2 weeks ago. I suspect any other modern Linux distro will provide nearly identical results.

The short summary is:

1. ​	Read /etc/resolv.conf
2. ​	Try to use nscd
3. ​	Try to use nscd again
4. ​	Read /etc/nsswitch.conf
5. ​	Load libnss_files.so
6. ​	Read /etc/host.conf
7. ​	Try to find IPv6 address in /etc/hosts
8. ​	Load libnss_dns.so
9. ​	Load libresolv.so
10. ​	Perform DNS IPv6 ‘AAAA’ query
11. ​	Try to find IPv4 address in /etc/hosts
12. ​	Perform DNS IPv4 ‘A’ query

Read on for the full trace with commentary.

```
strace -f getent hosts www.puppetlabs.com
...
open("/etc/resolv.conf", O_RDONLY)      = 3
...
close(3)                                = 0
```

Looking at the source for [GNU libc](http://ftp.gnu.org/gnu/glibc/) 2.5 (which is what is installed on this box), it appears that /etc/resolv.conf is loaded in resolv/res_init.c and the explanation is given as:

```
/*
 * Resolver state default settings.
 */

/*
 * Set up default settings.  If the configuration file exist, the values
 * there will have precedence.  Otherwise, the server address is set to
 * INADDR_ANY and the default domain name comes from the gethostname().
 *
 * An interrim version of this code (BIND 4.9, pre-4.4BSD) used 127.0.0.1
 * rather than INADDR_ANY ("0.0.0.0") as the default name server address
 * since it was noted that INADDR_ANY actually meant ``the first interface
 * you "ifconfig"'d at boot time'' and if this was a SLIP or PPP interface,
 * it had to be "up" in order for you to reach your own name server.  It
 * was later decided that since the recommended practice is to always
 * install local static routes through 127.0.0.1 for all your network
 * interfaces, that we could solve this problem without a code change.
 *
 * The configuration file should always be used, since it is the only way
 * to specify a default domain.  If you are running a server on your local
 * machine, you should say "nameserver 0.0.0.0" or "nameserver 127.0.0.1"
 * in the configuration file.
 *
 * Return 0 if completes successfully, -1 on error
 */
```

Okay. I guess. Let’s move on.

```
...
socket(PF_FILE, SOCK_STREAM, 0)         = 3
fcntl(3, F_SETFL, O_RDWR|O_NONBLOCK)    = 0
connect(3, {sa_family=AF_FILE, path="/var/run/nscd/socket"...}, 110) = -1 ENOENT
(No such file or directory)
close(3)                                = 0
socket(PF_FILE, SOCK_STREAM, 0)         = 3
fcntl(3, F_SETFL, O_RDWR|O_NONBLOCK)    = 0
connect(3, {sa_family=AF_FILE, path="/var/run/nscd/socket"...}, 110) = -1 ENOENT
(No such file or directory)
close(3)                                = 0
```

Why did you check nscd twice?

GNU libc nscd/nscd_helper.c is the only place with a connect() call referencing /var/run/nscd/socket (aka _PATH_NSCDSOCKET as defined in nscd/nscd-client.h). The connect() is in open_socket(), which is referenced in two places:

One:

```
/* Try to get a file descriptor for the shared memory segment
   containing the database.  */
static struct mapped_database *
get_mapping (request_type type, const char *key,
             struct mapped_database **mappedp)
```

Two:

```
/* Create a socket connected to a name. */
int
__nscd_open_socket (const char *key, size_t keylen, request_type type,
                    void *response, size_t responselen)
```

Here I took it upon myself to try to build the GNU libc code I was referencing. I figured I’d build it with debug symbols and then run getent again under gdb. The build with CFLAGS=-g spit out an error saying that it **must** be built with optimization. So much for that, but I did at least throw in some syslog() calls. For one, the two attempts to connect to an nscd socket above are in fact from both referenced functions.

**Update 1/11/2011: This shows my lack of gdb knowledge. One doesn’t need to build in debugging symbols to see what I am trying to see. Commenter Dave W. shows that below with his traces.**

```
Jan  3 03:57:33 new-host-2 getent: get_mapping() trying to open nscd socket with
open_socket()
Jan  3 03:57:33 new-host-2 getent: __nscd_open_socket() trying to open nscd socket
with open_socket() with open_socket()
```

Is that correct behavior? Could it be better? Beats me. I’m only taking it that far, but it doesn’t seem ideal.

```
...
open("/etc/nsswitch.conf", O_RDONLY)    = 3
...
close(3)                                = 0
```

Now we actually get somewhere. At least we’re reading the right configuration file at this point.

This is generated from GNU libc nss/nsswitch.c

```
int
__nss_database_lookup (const char *database, const char *alternate_name,
                       const char *defconfig, service_user **ni)
{
...
    service_table = nss_parse_file (_PATH_NSSWITCH_CONF);
```

Fine, moving on.

```
open("/lib64/libnss_files.so.2", O_RDONLY) = 3
...
close(3)                                = 0
```

This is due to “files” being first in /etc/nsswitch.conf. Fine.

```
...
open("/etc/host.conf", O_RDONLY)        = 3
...
close(3)                                = 0
```

The hell? You already found a valid /etc/nsswitch.conf. Why would you query this stupid old legacy file?

nss/getXXbyYY_r.c causes this read of /etc/host.conf

```
#ifdef NEED__RES_HCONF
          if (!_res_hconf.initialized)
            _res_hconf_init ();
#endif /* need _res_hconf */
```

Turns out this is hardcoded and not managed/overriden in any way by configure.

```
[jblaine@new-host-2 glibc-2.5]$ grep "#define NEED__RES_HCONF" */*
inet/gethstbyad_r.c:#define NEED__RES_HCONF     1
inet/gethstbynm2_r.c:#define NEED__RES_HCONF    1
inet/gethstbynm_r.c:#define NEED__RES_HCONF     1
```

***\*???\**** – feel free to provide a comment on this below. I don’t understand the need for this nowadays when we have /etc/nsswitch.conf.

```
...
open("/etc/hosts", O_RDONLY)            = 3
...
close(3)                                = 0
```

Makes sense finally, at least if this was the result of doing what our /etc/nsswitch.conf said (“files dns”).

**Update 1/11/2011: Oddly, this first opening of /etc/hosts is due to trying to resolve www.puppetlabs.com via an IPv6 address.**

```
open("/lib64/libnss_dns.so.2", O_RDONLY) = 3
...
close(3)                                = 0
...
open("/lib64/libresolv.so.2", O_RDONLY) = 3
...
close(3)                                = 0
```

Fine.

```
socket(PF_INET, SOCK_DGRAM, IPPROTO_IP) = 3
connect(3, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("192.168.1.1")}, 28) = 0
...
sendto(3, "uw\1\0\0\1\0\0\0\0\0\0\3www\npuppetlabs\3com\0"..., 36, MSG_NOSIGNAL, NULL, 0) = 36
...
recvfrom(3, "uw\201\200\0\1\0\1\0\1\0\0\3www\npuppetlabs\3com\0"..., 1024, 0,
{sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("192.168.1.1")}, [16]) = 120
close(3)                                = 0
```

DNS traffic, finally.

**Update 1/10/2011: After coming back to this little exercise tonight armed with Wireshark, I’ve found that this DNS request is for an IPv6 “AAAA” record. Commenter Dave W. confirmed this below. Again, this is odd to me that it would try IPv6 first.**

```
open("/etc/hosts", O_RDONLY)            = 3
...
close(3)                                = 0
```

Why? What did this and what is the reason?

**Update 1/11/2011: This is the attempt to look it up as an IPv4 address. The following lack of expected syslog() output is still a bit mysterious though.**

Opening /etc/hosts happens in 2 GNU libc functions:

One:

```
void
_sethtent(f)
        int f;
{
        if (!hostf)
                hostf = fopen(_PATH_HOSTS, "r" );
        else
                rewind(hostf);
        stayopen = f;
}
```

Two:

```
struct hostent *
_gethtent()
{
...
        if (!hostf && !(hostf = fopen(_PATH_HOSTS, "r" ))) {
                __set_h_errno (NETDB_INTERNAL);
                return (NULL);
        }
...
```

Let’s assume our “problem” is _gethtent(). It’s referenced 3 places:

One:

```
struct hostent *
_gethtbyname2(name, af)
        const char *name;
        int af;
```

Two:

```
struct hostent *
_gethtbyaddr(addr, len, af)
        const char *addr;
        size_t len;
        int af;
```

Three:

```
struct hostent *
gethostent()
```

Oddly, with plenty of syslog() calls in _sethtent() and _gethtent() around where the fopen() of /etc/hosts happens, I cannot get them to be reached. This odd opening of /etc/hosts remains a mystery.

Moving on.

```
...
socket(PF_INET, SOCK_DGRAM, IPPROTO_IP) = 3
connect(3, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("192.168.1.1")}, 28) = 0
...
sendto(3, "\256\261\1\0\0\1\0\0\0\0\0\0\3www\npuppetlabs\3com\0"..., 36, MSG_NOSIGNAL, NULL, 0) = 36
...
recvfrom(3, "\256\261\201\200\0\1\0\2\0\0\0\0\3www\npuppetlabs\3com\0"..., 1024, 0,
{sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("192.168.1.1")}, [16]) = 66
close(3)                                = 0
...
write(1, "74.207.250.144  puppetlabs.com w"..., 5074.207.250.144  puppetlabs.com
www.puppetlabs.com) = 50
exit_group(0)                           = ?
```

Another DNS query before we get our screen output and getent exits. ***\*Why?\****

**Update 1/11/2011: This is the IPv4 query of an “A” record finally.**

Feel free to chime in.