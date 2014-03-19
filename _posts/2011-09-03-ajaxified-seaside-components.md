---
layout: post
title: "Ajaxified Seaside Components"
description: ""
category: 
tags: []
---
{% include JB/setup %}

<div style="text-align: justify;">
<span class="Apple-style-span" style="font-family: inherit;">The&nbsp;<a href="http://www.seaside.st/">Seaside</a>&nbsp;sprint after the&nbsp;<a href="http://www.esug.org/">ESUG</a>&nbsp;conference in Edinburgh was just the perfect moment for implementing some of the ideas that have been floating around in my mind. Here's&nbsp;<a href="http://lists.squeakfoundation.org/pipermail/seaside/2011-June/026850.html">one I posted on the seaside mailinglist</a>, about which&nbsp;I was reminded by Nick Ager at the beginning of the sprint. Thanks for the reminder, Nick! ;-)</span></div>
<div style="text-align: justify;">
<span class="Apple-style-span" style="font-family: inherit;"><br />
</span></div>
<div style="text-align: justify;">
<span class="Apple-style-span" style="font-family: inherit; font-size: large;">What do I mean with Ajaxified Components?</span></div>
<div style="text-align: justify;">
<span class="Apple-style-span" style="font-family: inherit;">The standard way Seaside applications behave is to trigger a complete webpage rendering after you have performed an action (i.e. clicking a link that triggers a callback). That works just fine for many use cases but there are ample times where you do not want to trigger a complete page rendering and merely update those components that actually changed (i.e. those components that need to be rerendered on the user's web page).</span></div>
<div style="text-align: justify;">
<span class="Apple-style-span" style="font-family: inherit;">Using Ajax and <a href="http://jquery.com/">JQuery</a>, which are nicely integrated in Seaside, it is already fairly simple to implement such behavior. Nevertheless, the abstractions for such behavior as implemented by&nbsp;<a href="http://www.iliadproject.org/pages/Documentation/Introduction/Widgets">dirty widgets in Illiad</a>&nbsp;and the <a href="http://www.aidaweb.si/ajax.html">ajaxified web components in Aida</a>&nbsp;are very useful. Therefore, I decided to have a go at integrating the approach we had implemented for our <a href="http://www.yesplan.be/">Yesplan</a> application into Seaside itself, such that an "ajaxified Seaside components" implementation can be shared by different Seaside applications.</span></div>
<div style="text-align: justify;">
<span class="Apple-style-span" style="font-family: inherit;"><br />
</span></div>
<div style="text-align: justify;">
<span class="Apple-style-span" style="font-family: inherit; font-size: large;">How it works</span></div>
<div style="text-align: justify;">
<span class="Apple-style-span" style="font-family: inherit;">Using such ajaxified seaside components, the triggering of ajax updates for your application becomes as transparent as the full page rendering. Let us illustrate that by making an ajaxified counter application (sorry for the lack of an original example). The rendering method of the Counter component (shown below) uses a normal callback for the decrease action and an ajax callback for the increment action. The #callbackWithAjaxUpdate: selector accepts a Smalltalk block just like a normal callback. The difference is that after executing the callback block, Seaside will trigger the rendering of only those components that have been marked as "dirty". Marking a component as "dirty" is done by sending it the #markDirty message, which is exemplified in the increment callback.</span></div>
<blockquote>
<span class="Apple-style-span" style="font-family: 'Courier New', Courier, monospace; font-size: x-small;">html heading</span><br />
<span class="Apple-style-span" style="font-family: 'Courier New', Courier, monospace; font-size: x-small;"><span class="Apple-tab-span" style="white-space: pre;"> </span>level: 2;</span><br />
<span class="Apple-style-span" style="font-family: 'Courier New', Courier, monospace; font-size: x-small;"><span class="Apple-tab-span" style="white-space: pre;"> </span>with: value.</span><br />
<span class="Apple-style-span" style="font-family: 'Courier New', Courier, monospace; font-size: x-small;">html anchor</span><br />
<span class="Apple-style-span" style="font-family: 'Courier New', Courier, monospace; font-size: x-small;"><span class="Apple-tab-span" style="white-space: pre;"> </span>callbackWithAjaxUpdate: [ value := value + 1. self markDirty];</span><br />
<span class="Apple-style-span" style="font-family: 'Courier New', Courier, monospace; font-size: x-small;"><span class="Apple-tab-span" style="white-space: pre;"> </span>with: '+'.</span><br />
<span class="Apple-style-span" style="font-family: 'Courier New', Courier, monospace; font-size: x-small;">html space.</span><br />
<span class="Apple-style-span" style="font-family: 'Courier New', Courier, monospace; font-size: x-small;">html anchor</span><br />
<span class="Apple-style-span" style="font-family: 'Courier New', Courier, monospace; font-size: x-small;"><span class="Apple-tab-span" style="white-space: pre;"> </span>callback: [ value := value -1 ];</span><br />
<span class="Apple-style-span" style="font-family: 'Courier New', Courier, monospace; font-size: x-small;"><span class="Apple-tab-span" style="white-space: pre;"> </span>with: '-'.</span></blockquote>
<div style="text-align: justify;">
<span class="Apple-style-span" style="font-family: inherit;">That's it? Almost. Ajaxified Seaside components need to be a subclass of WAAjaxifiedSeasideComponent (which is itself a direct subclass of WAComponent). This is because any ajaxified component needs to hold on to an id that is the html id of its outermost html markup. The ajax update uses jQuery to replace that markup with the newly rendered one. Furthermore, it defines the standard ajax update script, which you sometimes need to override to perform additional behavior when the ajax update happens.</span></div>
<div style="text-align: justify;">
<span class="Apple-style-span" style="font-family: inherit;">If it's not an option for you to subclass from WAAjaxifiedComponent, it would suffice to just implement the few methods of that class into whatever component's implementation you want to ajaxify (or wait until traits are supported in all Smalltalk dialects).</span></div>
<div style="text-align: justify;">
<span class="Apple-style-span" style="font-family: inherit;">You are also not limited to anchor callbacks at all. In fact, it is really easy to embed the ajax update script in any other javascript you generate. For example, consider the following implementation of a reset button for the counter example. Sending the #ajaxifiedUpdateScriptWith: message to the canvas returns javascript that triggers the ajax update in exactly the same way as when using the #callbackWithAjaxUpdate: message on an anchor (the unary message #ajaxifiedUpdateScript exists as well).</span></div>
<blockquote>
<span class="Apple-style-span" style="font-family: 'Courier New', Courier, monospace; font-size: x-small;">html button<br /><span class="Apple-style-span" style="white-space: pre;">&nbsp;  </span>onClick: &nbsp;(html ajaxifiedUpdateScriptWith: [value := 0. self markDirty]);<br /><span class="Apple-style-span" style="white-space: pre;">&nbsp;  </span>with: 'Reset']]</span></blockquote>
<div style="text-align: justify;">
I should add that there is little magic to this implementation. I was even pleasantly surprised how easy it was to integrate this idea into the Seaside implementation itself. The #callbackWithAjaxUpdate: method is only a wrapper for a jQuery script callback and the gathering of all update scripts of all components in the application is done using subclasses of the standard Seaside component traversal visitors.</div>
<div style="text-align: justify;">
<span class="Apple-style-span" style="font-family: inherit;"><br />
</span></div>
<div style="text-align: justify;">
<span class="Apple-style-span" style="font-family: inherit; font-size: large;">Try it yourself</span></div>
<div style="text-align: justify;">
<span class="Apple-style-span" style="font-family: inherit;">The implementation and an expanded counter example are available on squeaksource3:&nbsp;<a href="http://ss3.gemstone.com/ss/SeasideAjaxifiedComponents.html">http://ss3.gemstone.com/ss/SeasideAjaxifiedComponents.html</a></span></div>
<div style="text-align: justify;">
<span class="Apple-style-span" style="font-family: inherit;">The current version is a dump of what Andy Kellens and I produced during the sprint. I think some pondering over naming and some cleanups are still to be expected.</span></div>
<div style="text-align: justify;">
<span class="Apple-style-span" style="font-family: inherit;">I am also aware that similar implementations have been done <a href="http://forum.world.st/Repainting-during-AJAX-callbacks-td2526051.html">before</a>, and, as I mentioned, this one is based upon how we do it in <a href="http://www.yesplan.be/">Yesplan</a>. Three elements are important here: the ability to specialize the update script on a per-component basis, the explicit triggering of ajax updates and the ability to embed such an update in your own generated javascripts. Since this approach seems to be what other Smalltalk web frameworks (such as <a href="http://www.iliadproject.org/">Illiad</a> and <a href="http://www.aidaweb.si/">Aida</a>) provide, I hope it might be useful for others to use, improve and extend.</span></div>
