# PyDoSc - Passive Google Dork (OSint) and Scrap

## Introduction

`pydosc` automates Google searching for potentially vulnerable web pages and applications on the Internet.  It replaces
manually performing Google dork searches with a web GUI browser.

There are 2 parts.  The first is `ghdb_scraper.py` that retrieves the latest Google dorks and the second portion is
`pydosc.py` that leverages the information gathered by `ghdb_scraper.py`.

The core Google search library now uses the more flexible [yagooglesearch](https://github.com/opsdisk/yagooglesearch)
instead of [googlesearch](https://github.com/MarioVilas/googlesearch).  Check out the
[yagooglesearch README](https://github.com/opsdisk/yagooglesearch/blob/master/README.md) for a more in-depth explanation
of the library differences and capabilities.

This version of `pydosc` also supports native HTTP(S) and SOCKS5 application support, so no more wrapping it in a tool
like `proxychains4` if you need proxy support.  You can specify multiple proxies to use in a round-robin fashion by
providing a comma separated string of proxies using the `-p` switch.

## What are Google dorks?

Offensive Security maintains the Google Hacking Database (GHDB) found here:
<https://www.exploit-db.com/google-hacking-database>.  It is a collection of Google searches, called dorks, that can be
used to find potentially vulnerable boxes or other juicy info that is picked up by Google's search bots.

## Terms and Conditions

The terms and conditions for `pydosc` are the same terms and conditions found in
[yagooglesearch](https://github.com/opsdisk/yagooglesearch#terms-and-conditions).

This code is supplied as-is and you are fully responsible for how it is used.  Scraping Google Search results may
violate their [Terms of Service](https://policies.google.com/terms).  Another Python Google search library had some
interesting information/discussion on it:

* [Original issue](https://github.com/aviaryan/python-gsearch/issues/1)
* [A response](https://github.com/aviaryan/python-gsearch/issues/1#issuecomment-365581431>)
* Author created a separate [Terms and Conditions](https://github.com/aviaryan/python-gsearch/blob/master/T_AND_C.md)
* ...that contained link to this [blog](https://benbernardblog.com/web-scraping-and-crawling-are-perfectly-legal-right/)

Google's preferred method is to use their [API](https://developers.google.com/custom-search/v1/overview).

## Installation

Scripts are written for Python 3.6+.  Clone the git repository and install the requirements.

```bash
git clone https://github.com/opsdisk/pydosc.git
cd pydosc
virtualenv -p python3.7 .venv  # If using a virtual environment.
source .venv/bin/activate  # If using a virtual environment.
pip install -r requirements.txt
```

## ghdb_scraper.py

To start off, `pydosc.py` needs a list of all the current Google dorks.  The repo contains a `dorks/` directory with
the current dorks when the `ghdb_scraper.py` was last run. It's advised to run `ghdb_scraper.py` to get the freshest
data before running `pydosc.py`.  The `dorks/` directory contains:

* the `all_google_dorks.txt` file which contains all the Google dorks, one per line
* the `all_google_dorks.json` file which is the JSON response from GHDB
* Individual category dorks

Dork categories:

```python
categories = {
    1: "Footholds",
    2: "File Containing Usernames",
    3: "Sensitives Directories",
    4: "Web Server Detection",
    5: "Vulnerable Files",
    6: "Vulnerable Servers",
    7: "Error Messages",
    8: "File Containing Juicy Info",
    9: "File Containing Passwords",
    10: "Sensitive Online Shopping Info",
    11: "Network or Vulnerability Data",
    12: "Pages Containing Login Portals",
    13: "Various Online devices",
    14: "Advisories and Vulnerabilities",
}
```

### Using ghdb_scraper.py as a script

Write all dorks to `all_google_dorks.txt`, `all_google_dorks.json`, and individual categories if you want more
contextual data about each dork.

```bash
python ghdb_scraper.py -s -j -i
```

### Using ghdb_scraper as a module

The `ghdb_scraper.retrieve_google_dorks()` function returns a dictionary with the following data structure:

```python
ghdb_dict = {
    "total_dorks": total_dorks,
    "extracted_dorks": extracted_dorks,
    "category_dict": category_dict,
}
```

Using a Python shell (like `python` or `ipython`) to explore the data:

```python
import ghdb_scraper

dorks = ghdb_scraper.retrieve_google_dorks(save_all_dorks_to_file=True)
dorks.keys()
dorks["total_dorks"]

dorks["extracted_dorks"]

dorks["category_dict"].keys()

dorks["category_dict"][1]["category_name"]
```

## <span>pydosc.py</span>

### Using <span>pydosc.py</span> as a script

```bash
python pydosc.py -d example.com -g dorks.txt 
```

### Using pydosc as a module

The `pydosc.pydosc.go()` function returns a dictionary with the data structure below (dorks used are made up examples):

```python
{
    "dorks": {
        "inurl:admin": {
            "urls_size": 3,
            "urls": [
                "https://github.com/marmelab/ng-admin",
                "https://github.com/settings/admin",
                "https://github.com/akveo/ngx-admin",
            ],
        },
        "inurl:gist": {
            "urls_size": 3,
            "urls": [
                "https://gist.github.com/",
                "https://gist.github.com/index",
                "https://github.com/defunkt/gist",
            ],
        },
    },
    "initiation_timestamp": "2021-08-27T11:35:30.638705",
    "completion_timestamp": "2021-08-27T11:36:42.349035",
}
```

Using a Python shell (like `python` or `ipython`) to explore the data:

```python
import pydosc

pg = pydosc.pydosc(
    google_dorks_file="dorks.txt",
    domain="github.com",
    max_search_result_urls_to_return_per_dork=3,
    save_pydosc_results_to_json_file=True,
    save_urls_to_file=True,
    verbosity=5,
)
pydosc_results_dict = pg.go()

pydosc_results_dict.keys()

pydosc_results_dict["initiation_timestamp"]

pydosc_results_dict["completion_timestamp"]

for key,value in pydosc_results_dict["dorks"].items():
    print(f"dork: {key}")
    for url in value["urls"]:
        print(url)
```

## Tuning Results

## Scope to a specific domain

The `-d` switch can be used to scope the results to a specific domain and functions as the Google search operator:

```none
site:github.com
```

### Wait time between Google dork searchers

* `-i` - Specify the **minimum** delay between dork searches, in seconds.  Don't make this too small, or your IP will
get HTTP 429'd quickly.
* `-x` - Specify the **maximum** delay between dork searches, in seconds.  Don't make this too big or the searches will
take a long time.

The values provided by `-i` and `-x` are used to generate a list of 20 randomly wait times, that are randomly selected
between each different Google dork search.

### Number of results to return

`-m` - The total max search results to return per Google dork.  Each Google search request can pull back at most 100
results at a time, so if you pick `-m 500`, 5 separate search queries will have to be made for each Google dork search,
which will increase the amount of time to complete.

## Google is blocking me!

Performing 7300+ search requests to Google as fast as possible will simply not work.  Google will rightfully detect it
as a bot and block your IP for a set period of time.  One solution is to use a bank of HTTP(S)/SOCKS proxies and pass
them to `pydosc`

### Native proxy support

Pass a comma separated string of proxies to `pydosc` using the `-p` switch.

```bash
python pydosc.py -g dorks.txt -p http://myproxy:8080,socks5h://127.0.0.1:9050,socks5h://127.0.0.1:9051
```

You could even decrease the `-i` and `-x` values because you will be leveraging different proxy IPs.  The proxies passed
to `pydosc` are selected by round robin.

### proxychains4 support

Another solution is to use `proxychains4` to round robin the lookups.

Install `proxychains4`

```bash
apt install proxychains4 -y
```

Edit the `/etc/proxychains4.conf` configuration file to round robin the look ups through different proxy servers.  In
the example below, 2 different dynamic socks proxies have been set up with different local listening ports (9050 and
9051).

```bash
vim /etc/proxychains4.conf
```

```ini
round_robin
chain_len = 1
proxy_dns
remote_dns_subnet 224
tcp_read_time_out 15000
tcp_connect_time_out 8000
[ProxyList]
socks4 127.0.0.1 9050
socks4 127.0.0.1 9051
```

Throw `proxychains4` in front of the `pydosc.py` script and each *request* lookup will go through a different proxy (and
thus source from a different IP).

```bash
proxychains4 python pydosc.py -g dorks/all_google_dorks.txt -o -s
```

Note that this may not appear natural to Google if you:

1) Simulate "browsing" to `google.com` from IP #1
2) Make the first search query from IP #2
3) Simulate clicking "Next" to make the second search query from IP #3
4) Simulate clicking "Next to make the third search query from IP #1

For that reason, using the built in `-p` proxy support is preferred because, as stated in the `yagooglesearch`
documentation, the "provided proxy is used for the entire life cycle of the search to make it look more human, instead
of rotating through various proxies for different portions of the search."

## Reference for Dorks

https://securitytrails.com/blog/google-hacking-techniques

## License

Distributed under the GNU General Public License v3.0. See [LICENSE](./LICENSE) for more information.

## Contact

Twitter: [@Cyberkid Adithya](https://twitter.com/CyberkidA)

LinkedIn: [@Adithya E S](https://www.linkedin.com/in/adithyaes)

Project Link: [https://github.com/CyberkidAdithya/PyDoSc](https://github.com/opsdisk/pydosc)
