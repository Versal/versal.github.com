# Sandbox

*November 30, 2012*

*Nick*

Sometimes user-provided javascript needs to be run, which is scary ([Evil.js anybody?](https://github.com/kitcambridge/evil.js)). At first, we tried to sanitize third-party code to make it safe, but this got ugly quickly. It meant that we had to prepend code to every CSS selector to make sure external styles only affected the right elements, the HTML had to always be well-formed, and let's not even talk about the JS. It also meant we couldn't surface meaningful javascript errors when something went wrong, because it was a small piece of javascript compiled in a larger application. We looked for another solution.

[JSFiddle](http://jsfiddle.net) and [JSBin](http://jsbin.com) seemed to have similar requirements, and handled it using iframes. As frontend developers though, our gut  aversion to iframes stopped us from typing the tag into our terminals in good conscience. However, they seemed to work nicely because the scripts couldn't touch anything but their box, CSS and JS selectors just worked, and we could monitor errors easily. [This](http://benvinegar.github.com/seamless-talk/) talk covers more of the benefits of iframes (and it's in 3D).

Although this isn't a very common requirement, we built a simple library to handle this which we call [Sandbox.js](http://github.com/versal/sandbox-js). It enables a few nice things like importing external JS and CSS as well as stopping code from creating dialogs (alert/confirm/prompt) unless desired. In the near future we hope to add error handling and a nice callback system so the frame can more easily communicate with the parent without nasty globals everywhere.

Happy Sandboxing!
