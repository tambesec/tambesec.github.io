
```
 findomain -t $target --quiet | tee findomain_subs.txt
```

```
 subfinder -d $target -all -recursive > subfinder_subs.txt -v all
```

```
 sublist3r -d $target -o sublist3r_subs.txt
```


```
assetfinder -subs-only $target | tee assetfinder_subs.txt
```
 
```
 amass enum -d mercadopago.com.ar | grep -oE '\b([a-zA-Z0-9_-]+\.)+mercadopago\.com\.ar\b' | sort -u
``` 

```
amass enum -d mercadopago.com.ar | grep '(FQDN)' | cut -d " " -f1 | anew amass_subs.txt
```

```
sed '/in-addr\.arpa$/d' amass_subs.txt > output.txt
```
 
```
 chaos -d $target -silent -o chaos_subs.txt -key YOUR_TOKEN
```

```
curl -s "https://crt.sh/?q=%25.volkswagen.de&output=json" | jq -r '.[].name_value' | sort -u > crtsh_subs.txt
```

```
github-subdomains -d $target -t YOUR_TOKEN -o github_subs.txt
```


```
subdominator -d $target -o subdominator_subs.txt
```
## Active recon subdomains

_Note: Chỉ dùng trong trường hợp không dùng wilcard dns (mọi subdomains đều trả về kết quả found) tránh làm nhiều thông tin của passive subdomains_

```
massdns -r /usr/share/wordlists/seclists/Miscellaneous/dns-resolvers.txt -t A -o S -w massdns_subs.txt /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt
```

```
shuffledns -d target.com -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt -r /usr/share/wordlists/seclists/Miscellaneous/dns-resolvers.txt -o shufflends_subs.txt -mode bruteforce -t 50
```


```
ffuf -u https://FUZZ.target.com -w /usr/share/wordlists/seclists/Discovery/DNS/bug-bounty-program-subdomains-trickest-inventory.txt -t 50 -mc 200,403 -o ffuf_subs.txt
```


```
ffuf -u "https://FUZZ.target.com" -w /usr/share/wordlists/seclists/Discovery/DNS/bug-bounty-program-subdomains-trickest-inventory.txt -t 50 -mc 200,403 -silent | \
awk -F '//' '{print $2}' | sort -u > ffuf_subs.txt
```

## More enumeration subdomains

- Shodan tools
```
shosubgo -d $target -s YOUR_TOKEN -o shodan_subs.txt
```

```
shosubgo -f all_subs.txt -s YOUR_TOKEN
```

- When we get this service/port information we can feed it to nmap to get a OG outputfile.

## Combine subdomains

```
cat *_subs.txt  | anew all_subdomains.txt
```

```
rm *_subs.txt
```


# Recon IP addresses

Shodan -> hostname:domain.com -> Top Products -> more -> IP

### Use code below to get IP

```
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

```
uncover -q “target.com” -e censys,fofa,shodan,shodan-idb | httpx | tee ips.txt
```

# Shortcut Urls

```
cat all_urls.txt | uro | tee -a shorcut_urls.txt
```
## Find live subdomain

```
cat all_subdomains.txt | httpx-toolkit -ports 443 -threads 200 > live_subdomains.txt
```

```
 cat all_subdomains.txt | httpx-toolkit -ports 80,443,8080,8000,8888 -threads 200 > live_subdomains.txt
```

# Check subdomain takeover

```
 subzy run --targets all_subdomains.txt --concurrency 100 --hide_fails --verify_ssl
```

```
 subzy run --targets live_subdomains.txt
```
# Screenshotting Web Services

```
httpx -l live_subdomains.txt -status-code -title -tech-detect -follow-redirects -o live_status.txt
```

```
cat list.txt | aquatone -out screenshots_aquatone
```

```
gowitness scan file -f probed_domains.txt --threads 10 --screenshot-path screenshots_gowitness --write-db
```

```
gowitness file -f all_subdomains.txt
```

# Link Discovery

### `Gospider`

```
gospider -s <target>
```


- Run with single site
```
gospider -s "https://domain.com" -c 10 -d 1
```

- Run with 20 sites & 10 bots each site 
```
gospider -S sites.txt -o output -c 10 -d 1 -t 20
```

# Conten Hidden

```
dirsearch -w /usr/share/wordlists/custom.txt --full-url --random-agent -x 404,400 -e php,html,js,json,ini -u https://target.com/
```

```
dirsearch -e php,asp,aspx,jsp,py,txt,conf,config,bak,backup,swp,old,db,sql,asp,aspx,asp~,py~,rb,rb~,php~,bak,bkp,cache,cgi,conf,csv,html,inc,jar,js,json,jsp~,lock,log,rar,old,sql.gz,sql.zip,sql.tar.gz,sql~,swp~,tar,tar.bz2,tar.gz,txt,wadl,zip -i 200 --full-url --deep-recursive -w /usr/share/wordlists/custom.txt --exclude-subdirs .well-known/,wp-includes/,wp-json/,faq/,Company/,Blog/,Careers/,Contact/,About/,IMAGE/,Images/,Logos/,Videos/,feed/,resources/,banner/,assets/,css/,fonts/,img/,images/,js/,media/,static/,templates/,uploads/,vendor/ --exclude-sizes 0B --skip-on-status 429 --random-agent -u http://target.com/
```

```
cat probed_domains.txt | feroxbuster --stdin -s 200 --no-recursion -k --random-agent --no-state -r -W 0 -w /usr/share/wordlists/custom.txt
```

```
ffuf -w /usr/share/wordlists/custom.txt -t 75 -ac -mc 200,405,401,415,302,301 -u http://assets.engage.tesla.com/FUZZ
```

```
xargs -a probed_domains.txt -I@ sh -c 'gobuster dir -u "@" --no-error -f -q -k -e --random-agent -w /usr/share/wordlists/custom.txt' | tee -a gobuster.txt
```


# Find information disclose

```
dirsearch -u domain -t 50 -x 429,404,502,503,400,500,403,428 -e conf,config,bak,backup,swp,old,db,sql,asp,aspx,aspx~,asp~,py,py~,rb,rb~,php,php~,bkp
```


```
dirsearch -e php,asp,aspx,jsp,py,txt,conf,config,bak,backup,swp,old,db,sql,asp,aspx,asp~,py~,rb,rb~,php~,bak,bkp,cache,cgi,conf,csv,html,inc,jar,js,json,jsp~,lock,log,rar,old,sql.gz,sql.zip,sql.tar.gz,sql~,swp~,tar,tar.bz2,tar.gz,txt,wadl,zip -i 200 --full-url --deep-recursive -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-files.txt --exclude-subdirs .well-known/,wp-includes/,wp-json/,faq/,Company/,Blog/,Careers/,Contact/,About/,IMAGE/,Images/,Logos/,Videos/,feed/,resources/,banner/,assets/,css/,fonts/,img/,images/,js/,media/,static/,templates/,uploads/,vendor/ --exclude-sizes 0B --skip-on-status 429 --random-agent -u https://www.mercadopago.com.mx/
```
# Vulnerability Scanners

```
nuclei -l live_subdomains.txt -t ~/.local/nuclei-templates/ -H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36' -s critical,high,medium,low -headless -code -dast -file -esc
```

```
nuclei -u https://target.com -headless -code -dast -file -esc
```

```
nuclei -u https://promociones.mercadopago.com.ar -headless -code -dast -file -esc -severity low,medium,high,critical -tags xss,sqli,lfi,rce,exposure
```

```
wpscan --url https://promociones.mercadopago.com.ar --api-token=YOUR_TOKEN
```

```
joomscan --url domain.com
```

```
nikto --url domain.com
```

```
mkdir nikto_result
cat all_subdomains.txt | xargs -I {} nikto -h {} -o nikto_scan_{}.log
```

```
cp live_subdomains.txt standardize_subdomains.txt
```

```
sudo osmedeus scan -T standardize_subdomains.txt
```

```
sudo osmedeus scan -m brutefocing-subdomain -t mercadopago.com.co
```

```
echo mercadopago.com.co | cariddi
```

```
misconfig-mapper -target "yourcompanyname" -service "*" -delay 1000
```

```
python3 greaper.py -l all_urls.txt -cors
```

### _Search online for the exploits across all the most popular collections: _Exploit-DB_, _Metasploit_, _Packetstorm_ and others_

```
getsploit --api-key YOUR_NEW_API_KEY 'jQuery'
```

### check websites for Code Injection and Server-Side Template Injection vulnerabilities

```
./sstimap.py -u https://example.com/page?name=John
```

# Url collections from file all subdomains.


```
cat urls.txt | cariddi -intensive
```


```
cat live_subdomains.txt | gau --threads 20 | anew all_urls.txt
```

```
katana -list all_subdomains.txt  -d 5 -jc -silent | anew all_urls.txt
```

```
cat all_subdomains.txt | hakrawler | anew all_urls.txt
```

```
 cat all_endpoints.txt | httpx-toolkit -status-code -mc 200,403,401,500 | anew sensitive_endpoints.txt
```
#  Archived URLs (one domain only)

`anew: **giúp quản lý dữ liệu thu thập được một cách hiệu quả**, tránh trùng lặp khi chạy nhiều lệnh liên tiếp.`

```
gau target.com | anew all_urls.txt
```

```
katana -u target.com -d 5  | anew all_urls.txt
```

_Multi threading_
```
parallel -j 2 '{} target.com | anew all_urls.txt' ::: gau katana
```

```
 wc -l all_urls.txt.txt
```



grep -v -F -x -f file1.txt file2.txt > result.txt
# JS Recon methodology 

```
cat live_subdomains.txt | katana -jc -kf js -silent | grep '\.js$' | anew all_js.txt

```

```
grep -E '\.js$|\.js\?' all_urls.txt > all_js.txt
```

```
grep '.js$' all_urls.txt | anew all_js.txt
```

```
 wc -l all_js.txt
```

```
cat all_js.txt | httpx-toolkit -mc 200 -o live_js.txt
```

```
 wc -l live_js.txt
```

```
cat live_js.txt | jsleak -s -l -k
```

```
cat live_js.txt | nuclei -t ~/nuclei-templates/credentials-disclosure-all.yaml -c 30
```

```
cat live_js.txt | nuclei -t ~/.local/nuclei-templates/http/exposures -c 30
```

```
source ~/tools/venv/bin/activate
```

```
cd ~/tools/lazyegg/
```

```
cat js_end.txt  | xargs -I{} bash -c 'echo -e "\ntarget : {}\n" && python lazyegg.py "{}" --js_urls --domains --ips --leaked_creds --local_storage'
```

```
javascript:(function(){var scripts=document.getElementsByTagName("script"),regex=/(?<=(\"|\'|\`))\/[a-zA-Z0–9_?&=\/\-\#\.]*(?=(\"|\'|\`))/g;const results=new Set;for(var i=0;i<scripts.length;i++){var t=scripts[i].src;""!=t&&fetch(t).then(function(t){return t.text()}).then(function(t){var e=t.matchAll(regex);for(let r of e)results.add(r[0])}).catch(function(t){console.log("An error occurred: ",t)})}var pageContent=document.documentElement.outerHTML,matches=pageContent.matchAll(regex);for(const match of matches)results.add(match[0]);function writeResults(){results.forEach(function(t){document.write(t+"<br>")})}setTimeout(writeResults,3e3);})();
```
### Find leak API key in js and html file

```
cat live_js.txt | mantra
```

```
cat live_js.txt | 
```

```
gitleaks detect all_urls.txt
```

# Filtering Interesting URLs


_Filter parameter_

cat all_urls.txt | httpx -silent -threads 100 | urldedupe -keep-params | tee live_unique_params.txt 

```
cat all_urls.txt | uro | tee -a shorcut_urls.txt
```


```
cat shorcut_urls.txt | gf xss | anew xss_candidates.txt 
cat shorcut_urls.txt | gf sqli | anew sqli_candidates.txt 
cat shorcut_urls.txt | gf lfi | anew lfi_candidates.txt 
```


```
grep -E "\.(env|conf|config|ini|yml|sql|db|sqlite|dump|log|txt|bak|old|swp|save|key|secret|token|password|aws|s3|gcp|azure|admin|debug|test|phpinfo)" all_urls.txt | sort -u > sensitive_urls.txt
```

```
grep -E "\.(bak|old|swp|save)" all_urls.txt
```

```
grep -E "\.(log|txt)" all_urls.txt | grep -v "robots.txt"
```

```
grep -E "(\.git/|\.svn/|\.hg/)" all_urls.txt
```

```
grep -E "(swagger|openapi|api-docs)" all_urls.txt
```

```
grep -E "(key|secret|token|password|credential|access|auth)" all_urls.txt | grep -v "\.js\|\.css"
```

```
grep -E "(aws_access_key|gcp_key|azure_secret)" all_urls.txt
```


```
echo "https://example.com/" | gau | gf lfi | uro | sed 's/=.*/=/' | qsreplace "FUZZ" | sort -u | xargs -I{} ffuf -u {} -w ~/payloads/lfi/lfi_payloads.txt -c -mr "root:(x|\*|\$[^\:]*):0:0:" -v
```

```
parallel -j 3 'cat gau_urls.txt | gf {} | anew {}_candidates.txt' ::: xss sqli lfi
```

# Open redirect

```
grep -Ei '(\?|&)(dest|redirect|uri|path|continue|url|window|next|data|reference|site|html|val|validate|domain|callback|return|page|feed|host|port|to|out|view|dir|show|navigation|open|signup_url|redirect_url|next_url|return_url|dest_url|path_url|callback_url|signup_referrer)=?' all_urls.txt
```

```
paramspider -l subdomain.txt > paramspider_redirect.txt

```

# Find command injection

```
commix --url="http://target.com/vuln.php?id=1" --skip-waf --batch
```



grep -Ei '(\?|&)(dest|redirect|uri|path|continue|url|window|next|data|reference|site|html|val|validate|domain|callback|return|page|feed|host|port|to|out|view|dir|show|navigation|open|signup_url|redirect_url|next_url|return_url|dest_url|path_url|callback_url|signup_referrer)=?' paramspider_redirect.txt > redirect_params.txt

sort *_redirect.txt > final_redirect.txt

wc -l final_redirect.txt
