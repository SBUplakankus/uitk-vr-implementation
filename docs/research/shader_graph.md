# Shader Graph in UI Toolkit

## Overview

As of Unity 6.3, Shader Graph Materials can now be applied directly to UITK Visual Elements to improve the graphical quality while maintaining the performance.

Some effects are still bad for VR and some effects like gradients should be done in USS, but for vector graphics backgrounds, scanlines, vignette, clouds, these can all be added now.

It is a very new implementation so resources are scarce but at least Unity are doing a big push promoting it right now. 



## Implementation

When generating UI Toolkit through code, it is very easy to quickly add effects and styles through a factory and the builder pattern which is what I have done in the past.

From what I have seen I should be able to get material implementation working alright. The main challenge will be finding what looks good without tanking performance too much. 

Did some tests and passed it back and forth with Claude Sonnet 4.5 and it told me that a styled panel with some shader effects layed on top would come in at > 0.15ms render time on the quest 2 since the shaders are all done through math and the base UITK Panel is only a single draw call combined.



## Resources

### YouTube Videos

-    [Shader Graph in UITK](https://www.youtube.com/watch?v=xAOBBW9hsjA)

- [Cam Ayres Vid (He Works for Unity)](https://www.youtube.com/watch?v=rW7qBt9i6Kg)


