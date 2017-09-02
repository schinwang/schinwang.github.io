### ansible

#### 1. installation

preparation:

{% highlight shell %}

$ sudo yum -y install openssl-devel wget python-devel
$ wget https://bootstrap.pypa.io/get-pip.py && sudo python get-pip.py
$ sudo pip install paramiko PyYAML Jinja2 httplib2 six

{% endhighlight %}

https://github.com/ansible/ansible下载稳定版

{% highlight shell %}

$ sudo python install.py install

{% endhighlight %}

ansible的module



