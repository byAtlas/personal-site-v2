+++
date = "2015-02-15T12:30:12+13:00"
draft = true
title = "Error Handling in ASP.NET MVC"

+++

So today we're gonna talk about how to make sure you don't show your users something like this: 

YSOD pic

While also making sure that you know that they would have been shown that, and hopefully give you some more infomation to boot. This post is quite long, since I've chosen to go through everything we need to add to the web.config (and why we need to add it). If you aren't really too worried about the reasoning, and trust me enough to just want to get to the money, there's a tldr riiiiiight down the bottom.

Not deterred? Good, we'll tackle the hiding bit first.

Ok, so at the moment, you're probably using something like this in your web.config:

	<customErrors mode="RemoteOnly">
		<error statusCode="404" path="~/Views/Errors/404.cshtml" />
		<error statusCode="500" path="~/Views/Errors/500.cshtml" />
	</customErrors>

Alternatively, you might be relying on a filter inside your MVC application to handle errors by showing an error.cshtml page or the like. <!--Discuss why this is a bad idea -->

So what exactly is going on here? The `mode="RemoteOnly"` means that these error pages will not get shown if you're browsing from localhost - reasonably useful for a dev environment. On the other hand, while you're doing these changes, you should probably set `mode` to `On`, meaning custom error pages are always shown. Otherwise you'll always see the yellow screen of death on your dev machine. Just remember to switch it all back afterwards, or all this effort will have been wasted when you deploy to production.

As the child elements we've got a bunch of individual definitions, mapping status codes to some views. Which only covers some of the cases, and somewhat poorly to boot. There's two issues - first off, if you end up on one of these pages via an error, and take a look in your network panel, you'll see that they both are actually returning redirects to something like `/Views/Errors/404?aspxerrorpath=/foo/bar`, and 200 then presenting a status code of 200, or "everythings ok". I'm not sure exactly why this is the default behaviour, but thankfully we can fix it.

First up, dealing with the status code that gets returned, making it a little more accurate. For these, you have to alter the actual template files themselves. If you're using razor, you can add this snippet near the top of the file:

	@{ Response.StatusCode = xxx }

If you're using old school asp, then you do the same thing, but pointier:

	<% Response.StatusCode = xxx %>

Ideally there'd be some way of jacking the error status code when getting directed from the custom error definition, but I haven't figured out a way of doing that yet, or if it's even possible. If anyone's got any ideas, hit me up at <a href="https://twitter.com/ByAtlas">@byAtlas</a>.

The other thing that's slightly concerning is that this still involves execution of your code. If you've got some hooks driven deep into your layouts or something that cause errors to be thrown, you won't get these error pages displayed correctly. <!-- Check this, and in what file types this sort of code can be executed. -->

Next up, to stop the redirects, add the `redirectMode="ResponseRewrite"` attribute to your customErrors tag:

	<customErrors mode="RemoteOnly" redirectMode="ResponseRewrite">
		<error statusCode="404" path="~/Views/Errors/404.cshtml" />
		<error statusCode="500" path="~/Views/Errors/500.cshtml" />
	</customErrors>

As for why exactly this isn't the default behaviour is a little beyond me. There's still a couple more things that need handling. Hopefully you've got a input handy somewhere you can try to shove an html tag into. You might notice this doesn't give you a one of your pretty little error pages. 

This is because it doesn't actually match any of the custom error pages we've defined, it will cause a 400 error - bad input. So the custom errors actually don't handle every single case under the sun. While we could add an exaustive list to the customErrors tag, there's another way around that will do for our purposes. 

So hopefully your error page is glouriously non-specific as to the nature of the error that occured - you don't really want to reveal to any potential attackers what exactly was happening that caused the server to freak out. So that being the case, we can probably just show the user the generic error page, by adding another attribute to the customErrors tag: `defaultRedirect="~/Views/Errors/500.cshtml"`, which should result in your custom errors section looking something like this:

	<customErrors mode="RemoteOnly" redirectMode="ResponseRewrite" defaultRedirect="~/Views/Errors/500.cshtml">
		<error statusCode="404" path="~/Views/Errors/404.cshtml" />
    	<error statusCode="500" path="~/Views/Errors/500.cshtml" />
	</customErrors>

In any case, this should ensure nothing makes it out of MVC with an ugly error page, or an incorrect status code. Unfortunately this still isn't enough. There are cases under which IIS will display error pages, rather than the error pages coming out of MVC.

So in order to deal with this, you have to add another section to your web.config, under the `system.webserver` tag:

	<httpErrors errorMode="Custom">
	  <remove statusCode="404"/>
	  <error statusCode="404" path="/404.html" responseMode="ExecuteURL"/>
	</httpErrors>