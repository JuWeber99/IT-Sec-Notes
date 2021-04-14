_**For convenience we can send it through Burpsuite to capture the traffic and let it automagically generate a site map for us.**_

# **Gobuster:**

### _**Proxy: `--proxy http://127.0.0.1:8080`**_

## Directories

-   `gobuster dir -u <url> -w <directoriy-wordlist-> -t <threads> -o gobuster/directory-base.txt -t 50`

## Files

-   `gobuster dir -u <url> -w <file-wordlist-> -t <threads> -o gobuster/files-base.txt -t 50 -x <extensions>`

### _**go recursive on found directories with both methodologies combindes**_

-   <# found dir 1>
    
-   <# found dir 2>
    

## Virtual Hosts:

-   `gobuster vhost -w <wordlist> -u <url for parent-domain to scan for subdomains/vhosts>`

## **FFUFF: TODO**

# **Dirbuster:**

## Directories in GUI toolset:

-   `dirbuster`

# **Wfuzz:**

_Normally for parameter bruteforcing or other fuzzable things. for directories and virtual hosts use gobuster_

basic usage: `wfuzz -w <wordlist> [<url>/FUZZ](http://testphp.vulnweb.com/FUZZ)`

More: [hacktricks cheatsheet](https://book.hacktricks.xyz/pentesting-web/web-tool-wfuzz "https://book.hacktricks.xyz/pentesting-web/web-tool-wfuzz")

### _Filters_

-   hide/show responses with length of given number
    
    -   `-- hl <number>`
    -   `-- sl <number>`
-   hide/show responses with regex:
    
    -   `--hs <"regex">`
    -   `--ss <"regex">`
-   hide/show by number of words:
    
    -   `--hw <number>`
    -   `--sw <number>`
-   hide/show responses with status code:
    
    -   `--hc <status codes>`
    -   `--sc <status codes>`

---