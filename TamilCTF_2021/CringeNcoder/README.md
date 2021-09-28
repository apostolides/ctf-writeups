# Tamil CTF 2021 - CringeNcoder
### Description:
Flag is located at flag, but its not flag.
### Solution:
Given a custom message encoder and an encoded message we must decode the message located at /flag. Encoding each allowed character and keeping a mapping can help us reverse the message.

```python
import requests as r
from bs4 import BeautifulSoup

url = "http://45.79.195.170:5000/encode"
chars = "0123456789abcdefghijklmnopqrstuvwxyz"
mapping = {}

print("[*] Creating map...")

for char in chars:
    payload = {'text':char}
    response = r.post(url,payload)
    if response.status_code == 200:
        soup = BeautifulSoup(response.text,'html.parser')
        for res in soup.select('.result > h1:nth-child(1) > a:nth-child(2)'):
            mapping[res.get_text().strip(' \n\t')] = char


print("[*] Map creation finished.")
print("[*] Decoding...")

encoded = "cR1Ng3e crinG3 cringE cringe cRINGe cRINGe cring3 crinG3 cringE cringE cRinG3 Cr1nGe cRimG3 criNG3 cRinge cRimG3 cringe cR1Ng3e cRiNge cRinG3 cR1Ng3 CrInGe cRInGE cr1ngE criNG3 cringE cring3 cRinge cR1Ng3 crinGE cringE criNgee cRinG3 CrInGe"
flag = "TamilCTF{"

for word in encoded.split():
    flag += mapping[word]

flag += "}"
print(F"[*] Flag is: {flag}")
print(mapping)

```
Happy hacking :)