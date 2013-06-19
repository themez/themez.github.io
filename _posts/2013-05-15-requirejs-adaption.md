---
layout: post
title: How did I screwed our project by adapting requirejs
category: tech
tags: [requirejs, javascript, AMD]
---

I've recently screwed our project, by introducing requirejs as a module loader. Now customers complain everyday, why XXX feature not works? Why it becomes so slow?

We have a [AMD](https://github.com/amdjs/amdjs-api/wiki/AMD) module loader before, which is developed by ourselves, it doesn't have so much features as requirejs, and the interfaces are sort of, naive, and verbose. The module dependencies is written in annotation, so modules have to be compiled before user can run it in browser, and the build process is not fast enough to feel comfortable.

I decided to make this changed, at first I tried to develop a new AMD module loader, but soon I realized that I could never make it better as requirejs, so I decided to adapt requirejs, I knew there're some differences between implementation of our loader and requirejs's, but I thought I can eliminate most of them by attaching some extra codes, and ask our customers to change client codes a little to eliminates the others. So I did.

After the migration been done, problems come, almost immediately. 

The first problem is that we used to expose our API as some global accessable object, such as `foo.bar2`, `foo.bar2`. But requirejs do not expose any modules to global namespace, however, I can expose modules by myself:

	(function(global){
		var exposedList = ["foo/bar1","foo/bar2"];
		
		require(exposedList, function(){
			//require modules in the list and put them on the corresponding namespaces.
			var exposedModules = arguments;
			exposedList.forEach(function(name, i){
				var nameParts = name.split("/"),
				    lastI = nameParts.length - 1
				nameParts.reduce(function(prev, cur, pi, arr){
					if(i === lastI){
						prev[cur] = exposedModules[i]
					}else if(!prev[cur]){
						prev[cur] = {}
					}
					return prev[cur];
				};
			}, global);
		});
	})(this);

So this is actually not a big problem, but there's a little problem that the callback function in `require` would not be called immediatly, even if all modules in `exposedList` already defined. So if client code call our API immediately, it's a problem.

I resolve this by hold DOM ready (`jQuery.holdReady(true)`) before I put all neccessary modules onto global namespace, and ask client to use our API after DOM ready.

	$(document).ready(function(){
		//client code
	});

OK, now we can see this problem resolved, but unfortunately there're more.

We resolve the previous async problem by hold DOM ready, but what if there's dynamic evaluating?

Let's think about this case, we're going to create a simple chart using our charting library, and we have some theme resources which are not included in the all-in-one js file, but in the form of pieces of javascript code, we'd like to apply one of the themes. So we include the theme file of "flashy" in a `script` tag.

	//flashy theme content
	require(["themeManager"],function(tm){
		tm.register("flashy", {...});
	});

Client assume that the theme of "flashy" should be available after the script loaded, but it's not. As we know, the callback function in `require` would not be evaluated immediately. So we won't be accessable to the "flashy" theme.

	<script src="flashy_theme.js"></script>
	<script>
		//fail!
		themeManager.get("flashy");
	</script>


To resolve this problem, we have to enforce our client to adapt requirejs too.

	<script src="flashy_theme.js"></script>
	<script>
		//this will work.
		require(["themeManager"], function(themeManager){
			themeManager.get("flashy");
		});
	</script>

But this is not we want, we've broken back compatibility.

Now we've decided, in short term, to use requirejs in development, and implement our own AMD module loader in release, which we have full control.

This is a fail case of adapting requirejs, I try to mix requirejs convention code and synchronous code together, it just won't work, I've underestimate the impact.