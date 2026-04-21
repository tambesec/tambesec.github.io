---
title: Bug Bounty Methodology
date: 2026-04-21 00:00:00 +0700
layout: post
categories:
  - Methodology
tags:
  - bug bounty
  - recon
  - pentest
---

This methodology was built out of need to automate and properly sequence the many tools required for bug bounty hunting and penetration testing.

## Subdomain Enumeration

```bash
findomain -t $target --quiet | tee findomain_subs.txt
```

```bash
subfinder -d $target -all -recursive > subfinder_subs.txt -v all
```

```bash
sublist3r -d $target -o sublist3r_subs.txt
```

```bash
assetfinder -subs-only $target | tee assetfinder_subs.txt
```
 
```bash
amass enum -d mercadopago.com.ar | grep -oE '\b([a-zA-Z0-9_-]+\.)+mercadopago\.com\.ar\b' | sort -u
``` 

```bash
amass enum -d mercadopago.com.ar | grep '(FQDN)' | cut -d " " -f1 | anew amass_subs.txt
```

```bash
sed '/in-addr\.arpa$/d' amass_subs.txt > output.txt
```
 
```bash
chaos -d $target -silent -o chaos_subs.txt -key YOUR_TOKEN
```

```bash
curl -s "https://crt.sh/?q=%25.volkswagen.de&output=json" | jq -r '.[].name_value' | sort -u > crtsh_subs.txt
```

```bash
github-subdomains -d $target -t YOUR_TOKEN -o github_subs.txt
```

```bash
subdominator -d $target -o subdominator_subs.txt
```

### Active Recon Subdomains


```bash
massdns -r /usr/share/wordlists/seclists/Miscellaneous/dns-resolvers.txt -t A -o S -w massdns_subs.txt /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt
```

```bash
shuffledns -d target.com -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt -r /usr/share/wordlists/seclists/Miscellaneous/dns-resolvers.txt -o shufflends_subs.txt -mode bruteforce -t 50
```

```bash
ffuf -u https://FUZZ.target.com -w /usr/share/wordlists/seclists/Discovery/DNS/bug-bounty-program-subdomains-trickest-inventory.txt -t 50 -mc 200,403 -o ffuf_subs.txt
```

```bash
ffuf -u "https://FUZZ.target.com" -w /usr/share/wordlists/seclists/Discovery/DNS/bug-bounty-program-subdomains-trickest-inventory.txt -t 50 -mc 200,403 -silent | \
awk -F '//' '{print $2}' | sort -u > ffuf_subs.txt
```

### More Enumeration Subdomains

- Shodan tools
```bash
shosubgo -d $target -s YOUR_TOKEN -o shodan_subs.txt
```

```bash
shosubgo -f all_subs.txt -s YOUR_TOKEN
```

- When we get this service/port information we can feed it to nmap to get a OG outputfile.

### Combine Subdomains

```bash
cat *_subs.txt  | anew all_subdomains.txt
rm *_subs.txt
```

## Recon IP Addresses

Shodan -> hostname:domain.com -> Top Products -> more -> IP

### Use code below to get IP

```javascript
var ipElements = document.querySelectorAll("strong");
var ips = [];

ipElements.forEach(function (e) {
    ips.push(e.innerHTML.replace(/["']/g, ""));
});

var ipsString = ips.join("\n");

var a = document.createElement("a");
a.href = "data:text/plain;charset=utf-8," + encodeURIComponent(ipsString);
a.download = "ips.txt";
document.body.appendChild(a);
a.click();
```

```bash
uncover -q “target.com” -e censys,fofa,shodan,shodan-idb | httpx | tee ips.txt
```

## Shortcut URLs

```bash
cat all_urls.txt | uro | tee -a shorcut_urls.txt
```

## Find Live Subdomains

```bash
cat all_subdomains.txt | httpx-toolkit -ports 443 -threads 200 > live_subdomains.txt
cat all_subdomains.txt | httpx-toolkit -ports 80,443,8080,8000,8888 -threads 200 > live_subdomains.txt
```

## Check Subdomain Takeover

```bash
subzy run --targets all_subdomains.txt --concurrency 100 --hide_fails --verify_ssl
subzy run --targets live_subdomains.txt
```

## Screenshotting Web Services

```bash
httpx -l live_subdomains.txt -status-code -title -tech-detect -follow-redirects -o live_status.txt
cat list.txt | aquatone -out screenshots_aquatone
gowitness scan file -f probed_domains.txt --threads 10 --screenshot-path screenshots_gowitness --write-db
gowitness file -f all_subdomains.txt
```

## Link Discovery

### Gospider

```bash
gospider -s <target>
```

- Run with single site
```bash
gospider -s "https://domain.com" -c 10 -d 1
```

- Run with 20 sites & 10 bots each site 
```bash
gospider -S sites.txt -o output -c 10 -d 1 -t 20
```

## Content Hidden (Directory Bruteforcing)

```bash
dirsearch -w /usr/share/wordlists/custom.txt --full-url --random-agent -x 404,400 -e php,html,js,json,ini -u https://target.com/
```

```bash
dirsearch -e php,asp,aspx,jsp,py,txt,conf,config,bak,backup,swp,old,db,sql,asp,aspx,asp~,py~,rb,rb~,php~,bak,bkp,cache,cgi,conf,csv,html,inc,jar,js,json,jsp~,lock,log,rar,old,sql.gz,sql.zip,sql.tar.gz,sql~,swp~,tar,tar.bz2,tar.gz,txt,wadl,zip -i 200 --full-url --deep-recursive -w /usr/share/wordlists/custom.txt --exclude-subdirs .well-known/,wp-includes/,wp-json/,faq/,Company/,Blog/,Careers/,Contact/,About/,IMAGE/,Images/,Logos/,Videos/,feed/,resources/,banner/,assets/,css/,fonts/,img/,images/,js/,media/,static/,templates/,uploads/,vendor/ --exclude-sizes 0B --skip-on-status 429 --random-agent -u http://target.com/
```

```bash
cat probed_domains.txt | feroxbuster --stdin -s 200 --no-recursion -k --random-agent --no-state -r -W 0 -w /usr/share/wordlists/custom.txt
```

```bash
ffuf -w /usr/share/wordlists/custom.txt -t 75 -ac -mc 200,405,401,415,302,301 -u http://assets.engage.tesla.com/FUZZ
```

```bash
xargs -a probed_domains.txt -I@ sh -c 'gobuster dir -u "@" --no-error -f -q -k -e --random-agent -w /usr/share/wordlists/custom.txt' | tee -a gobuster.txt
```


## Find Information Disclose

```bash
dirsearch -u domain -t 50 -x 429,404,502,503,400,500,403,428 -e conf,config,bak,backup,swp,old,db,sql,asp,aspx,aspx~,asp~,py,py~,rb,rb~,php,php~,bkp
```

```bash
dirsearch -e php,asp,aspx,jsp,py,txt,conf,config,bak,backup,swp,old,db,sql,asp,aspx,asp~,py~,rb,rb~,php~,bak,bkp,cache,cgi,conf,csv,html,inc,jar,js,json,jsp~,lock,log,rar,old,sql.gz,sql.zip,sql.tar.gz,sql~,swp~,tar,tar.bz2,tar.gz,txt,wadl,zip -i 200 --full-url --deep-recursive -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-files.txt --exclude-subdirs .well-known/,wp-includes/,wp-json/,faq/,Company/,Blog/,Careers/,Contact/,About/,IMAGE/,Images/,Logos/,Videos/,feed/,resources/,banner/,assets/,css/,fonts/,img/,images/,js/,media/,static/,templates/,uploads/,vendor/ --exclude-sizes 0B --skip-on-status 429 --random-agent -u https://www.mercadopago.com.mx/
```

## Vulnerability Scanners

```bash
nuclei -l live_subdomains.txt -t ~/.local/nuclei-templates/ -H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36' -s critical,high,medium,low -headless -code -dast -file -esc
```

```bash
nuclei -u https://target.com -headless -code -dast -file -esc
```

```bash
nuclei -u https://promociones.mercadopago.com.ar -headless -code -dast -file -esc -severity low,medium,high,critical -tags xss,sqli,lfi,rce,exposure
```

```bash
wpscan --url https://promociones.mercadopago.com.ar --api-token=YOUR_TOKEN
```

```bash
joomscan --url domain.com
nikto --url domain.com
```

```bash
mkdir nikto_result
cat all_subdomains.txt | xargs -I {} nikto -h {} -o nikto_scan_{}.log
```

```bash
cp live_subdomains.txt standardize_subdomains.txt
sudo osmedeus scan -T standardize_subdomains.txt
sudo osmedeus scan -m brutefocing-subdomain -t mercadopago.com.co
echo mercadopago.com.co | cariddi
misconfig-mapper -target "yourcompanyname" -service "*" -delay 1000
python3 greaper.py -l all_urls.txt -cors
```

### Search Online Exploits

*Search online for exploits across popular collections: Exploit-DB, Metasploit, Packetstorm, etc.*

```bash
getsploit --api-key YOUR_NEW_API_KEY 'jQuery'
```

### Check Websites for Code Injection and SSTI

```bash
./sstimap.py -u https://example.com/page?name=John
```

## URL Collections

```bash
cat urls.txt | cariddi -intensive
cat live_subdomains.txt | gau --threads 20 | anew all_urls.txt
katana -list all_subdomains.txt  -d 5 -jc -silent | anew all_urls.txt
cat all_subdomains.txt | hakrawler | anew all_urls.txt
cat all_endpoints.txt | httpx-toolkit -status-code -mc 200,403,401,500 | anew sensitive_endpoints.txt
```

## Archived URLs



```bash
gau target.com | anew all_urls.txt
katana -u target.com -d 5  | anew all_urls.txt
```

*Multi threading:*
```bash
parallel -j 2 '{} target.com | anew all_urls.txt' ::: gau katana
```

```bash
wc -l all_urls.txt
grep -v -F -x -f file1.txt file2.txt > result.txt
```

## JS Recon Methodology

```bash
cat live_subdomains.txt | katana -jc -kf js -silent | grep '\.js$' | anew all_js.txt
grep -E '\.js$|\.js\?' all_urls.txt > all_js.txt
grep '.js$' all_urls.txt | anew all_js.txt
wc -l all_js.txt
cat all_js.txt | httpx-toolkit -mc 200 -o live_js.txt
wc -l live_js.txt
cat live_js.txt | jsleak -s -l -k
```

```bash
cat live_js.txt | nuclei -t ~/nuclei-templates/credentials-disclosure-all.yaml -c 30
cat live_js.txt | nuclei -t ~/.local/nuclei-templates/http/exposures -c 30
```

```bash
source ~/tools/venv/bin/activate
cd ~/tools/lazyegg/
cat js_end.txt  | xargs -I{} bash -c 'echo -e "\ntarget : {}\n" && python lazyegg.py "{}" --js_urls --domains --ips --leaked_creds --local_storage'
```

### Find Leak API Key in JS and HTML

```bash
cat live_js.txt | mantra
gitleaks detect all_urls.txt
```

## Filtering Interesting URLs

*Filter parameter:*

```bash
cat all_urls.txt | httpx -silent -threads 100 | urldedupe -keep-params | tee live_unique_params.txt 
cat all_urls.txt | uro | tee -a shorcut_urls.txt
```

```bash
cat shorcut_urls.txt | gf xss | anew xss_candidates.txt 
cat shorcut_urls.txt | gf sqli | anew sqli_candidates.txt 
cat shorcut_urls.txt | gf lfi | anew lfi_candidates.txt 
```

```bash
grep -E "\.(env|conf|config|ini|yml|sql|db|sqlite|dump|log|txt|bak|old|swp|save|key|secret|token|password|aws|s3|gcp|azure|admin|debug|test|phpinfo)" all_urls.txt | sort -u > sensitive_urls.txt
grep -E "\.(bak|old|swp|save)" all_urls.txt
grep -E "\.(log|txt)" all_urls.txt | grep -v "robots.txt"
grep -E "(\.git/|\.svn/|\.hg/)" all_urls.txt
grep -E "(swagger|openapi|api-docs)" all_urls.txt
grep -E "(key|secret|token|password|credential|access|auth)" all_urls.txt | grep -v "\.js\|\.css"
grep -E "(aws_access_key|gcp_key|azure_secret)" all_urls.txt
```

```bash
echo "https://example.com/" | gau | gf lfi | uro | sed 's/=.*/=/' | qsreplace "FUZZ" | sort -u | xargs -I{} ffuf -u {} -w ~/payloads/lfi/lfi_payloads.txt -c -mr "root:(x|\*|\$[^\:]*):0:0:" -v
```

```bash
parallel -j 3 'cat gau_urls.txt | gf {} | anew {}_candidates.txt' ::: xss sqli lfi
```

## Open Redirect

```bash
grep -Ei '(\?|&)(dest|redirect|uri|path|continue|url|window|next|data|reference|site|html|val|validate|domain|callback|return|page|feed|host|port|to|out|view|dir|show|navigation|open|signup_url|redirect_url|next_url|return_url|dest_url|path_url|callback_url|signup_referrer)=?' all_urls.txt
paramspider -l subdomain.txt > paramspider_redirect.txt
```

## Find Command Injection

```bash
commix --url="http://target.com/vuln.php?id=1" --skip-waf --batch
```

```bash
grep -Ei '(\?|&)(dest|redirect|uri|path|continue|url|window|next|data|reference|site|html|val|validate|domain|callback|return|page|feed|host|port|to|out|view|dir|show|navigation|open|signup_url|redirect_url|next_url|return_url|dest_url|path_url|callback_url|signup_referrer)=?' paramspider_redirect.txt > redirect_params.txt
sort *_redirect.txt > final_redirect.txt
wc -l final_redirect.txt
```