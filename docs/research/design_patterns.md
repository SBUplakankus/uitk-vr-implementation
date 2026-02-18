# Design Patterns for UI Toolkit

## Design Patterns

### Factories

You can code the UXML Documents for specific static panels that won't change at run time. This is just like making a html page. For generating in C# it is best to use factories for visual element creation. All of the UXML Elements can be made through C# but it is alot of boilerplate so you can just ask AI to generate them for you and they will work perfectly. It's then easy to modify each of the functions and add variations for specific styles where needed. I use Claude Sonnet 4.5 on the free tier which is pretty good.

### Constants and References

The main downside to UITK right now is that it is ran on strings so a constants class is crucial for passing through styles in C#. I just make a UIToolkitStyles class to store them all in for type-safety. It is also good to cache the visual elements on creation so you don't need to run string queries to find the element when trying to update at runtime. 

### Builder

It'll also be good to implement the builder pattern to quickly and safely generate the panels in the C# view classes. The main problem I came across myself was all my view classes becoming repetetive long lists of Factory calls and adding to elements so this could be done much more smoothly on a bigger project. Would need to figure out the specifics but it'll still probably be better to create elements that need to be cached directly from the factory then add them, but the builder can easily set the containers, styles, text, images, shaders, etc.