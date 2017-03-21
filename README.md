# About this website

These are the files for generating [understandingdata.net](http://www.understandingdata.net).

I use [hugo](https://gohugo.io/) to make the site, which lets me keep almost all of the content in a simple set of [markdown](https://daringfireball.net/projects/markdown/) files. This is a slightly modified version of the [hyde theme](http://themes.gohugo.io/hyde/). 

To build the site, I just run

```{}
hugo
```
in the website's directory, and then upload all of the contents of the resulting `public` folder to my bucket on AWS.

I get my domain and email service through [namecheap](https://www.namecheap.com/) and the files are all on [AWS S3](https://aws.amazon.com/). 
