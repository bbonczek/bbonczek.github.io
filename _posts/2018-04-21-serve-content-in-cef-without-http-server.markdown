---
layout: post
title:  "serve content in CEF without http server"
date:   2018-04-18 21:44:09 +0200
categories: jekyll update
---
So, you want to serve page via `Chromium Embedded Framework`, but you don't want to host http server (or make client do it)? There are 2 ways to achieve it, but if you are only interested in quick solution, scroll down to `custom file protocol`.

### why you can't just use file protocol

If you are here, than you have propably already tried using `file://` protocol to make it work, but you have failed - you styles are not applied, or images don't show up. Why? Due to security restriction that comes with chromium when using file protocol. Let's say that you have bundled your web application into this structure:
{% highlight ruby %}
|- dist
    |- index.html
    |- bundle.js
    |- vendors.js
    |- styles.css
    |- assets
        |- arrow.svg
        |- start.svg
        |- stop.svg
{% endhighlight %}

and in your index.html you are requesting one of graphics located in `assets` folder.:

{% highlight html %}
...
<p>image from assets requested below:</p>
<img src="images/arrow.svg"/>
...
{% endhighlight %}

Image will not be rendered, because of error looking like this:

`XMLHttpRequest cannot load file:///.../images/arrow.svg. Cross origin requests are only supported for HTTP.`

Chromium will not allow to you load file that has different origin (basically resided in different folder). Why it happens is because directory tree `is not` treated as single origin. Anything that is a child, relative to your index.html (in this case) is considered to be different origin. To read more about it, you can check out [this chromium issue][chromium-isue].

### why `--allow-file-access-from-files` is not a good idea

Sure, running chrome with this flag will make you app work - unfortunatelly it will open your whole file system to malicious code.

Imagine that someone injected this into your code:

{% highlight html %}
...
<p>image from assets requested below:</p>
<img src="../../very-important-stuff-inside/passwords.png"/>
...
{% endhighlight %}

This snippet could easly reveal all you important stuff (keeping your passwords in png? Sure, why not).
That's why we need:

### custom file protocol

CEF allows you to specify custom protocols, that allow you to craft response sufficient for your needs. What we want to create is something simmilar to file protocol, but this something should allow you to work with files from whole directory tree. For purpouse of this article let's call it `custom protocol`. Let's get to work.

#### `CustomProtocolSchemeHandler.cs`

{% highlight C# %}
using System;
using System.IO;
using CefSharp;

namespace MyProject.CustomProtocol
{
    public class CustomProtocolSchemeHandler` : ResourceHandler
    {
        // Specifies where you bundled app resides.
        // Basically path to your index.html
        private string frontendFolderPath;

        public CustomProtocolSchemeHandler`()
        {
            frontendFolderPath = Path.Combine(AppDomain.CurrentDomain.BaseDirectory, "./bundle/");
        }

        // Process request and craft response.
        public override bool ProcessRequestAsync(IRequest request, ICallback callback)
        {
            var uri = new Uri(request.Url);
            var fileName = uri.AbsolutePath;

            var requestedFilePath = frontendFolderPath + fileName;

            var isAccesToFilePermitted = IsRequestedPathInsideFolder(
                new DirectoryInfo(requestedFilePath),
                new DirectoryInfo(frontendFolderPath));

            if (isAccesToFilePermitted && File.Exists(requestedFilePath))
            {
                byte[] bytes = File.ReadAllBytes(requestedFilePath);
                Stream = new MemoryStream(bytes);

                var fileExtension = Path.GetExtension(fileName);
                MimeType = GetMimeType(fileExtension);

                callback.Continue();
                return true;
            }

            callback.Dispose();
            return false;
        }

        // Added for security reasons.
        // In this code it is used to verify that requested file is descendant to your index.html.
        public bool IsRequestedPathInsideFolder(DirectoryInfo path, DirectoryInfo folder)
        {
            if (path.Parent == null)
            {
                return false;
            }

            if (string.Equals(path.Parent.FullName, folder.FullName, StringComparison.InvariantCultureIgnoreCase))
            {
                return true;
            }

            return IsRequestedPathInsideFolder(path.Parent, folder);
        }
    }
}
{% endhighlight %}

We are building our class basing on SchemeHandler.cs provided by Cef.

#### `CustomProtocolSchemeHandlerFactory.cs`

{% highlight C# %}
using CefSharp;

namespace MyProject.ExtendedFileProtocol
{
    public class CustomProtocolSchemeHandlerFactory : ISchemeHandlerFactory
    {
        public const string SchemeName = "customFileProtocol";

        public IResourceHandler Create(IBrowser browser, IFrame frame, string schemeName, IRequest request)
        {
            return new CustomProtocolSchemeHandler();
        }
    }
}
{% endhighlight %}

At this point we are almost good to go. Now go to place, where you call Cef.Initialize(...), and add this code:

#### `Control.xaml.cs`

{% highlight C# %}
/* ... */

/* ... */
{% endhighlight %}
.

.

.

.

.

.

.

.

.

.

.

You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets:

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')UA-117970144-1
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
[chromium-isue]: https://bugs.chromium.org/p/chromium/issues/detail?id=47416