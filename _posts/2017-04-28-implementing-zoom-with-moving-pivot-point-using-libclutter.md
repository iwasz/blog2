---
layout: post
title: Implementing zoom with moving pivot point using libclutter
permalink: http://www.iwasz.pl/uncategorized/implementing-zoom-with-moving-pivot-point-using-libclutter/index.html
post_id: 480
categories: 
- Uncategorized
---

I'm making this post, because I struggled with this functionality a lot! I was implementing it (to the extent I was happy with) in the course of 4 or even 5 days! Ok, first goes an animated gif which shows my desired zoom behavior (check out Inkscape, or other graphics programs, they all work the same in this regard):

[caption id="" align="alignnone" width="640"]
![](http://iwasz.pl/files/zoomok.gif) Desired zoom functionality[/caption]

Basically the idea is that, the center of the scale transformation is where the mouse pointer is on the screen, so the object under the cursor does not move while scaling, whereas other objects around it do. My first, and most obvious implementation (which didn't work as expected) was as such:/*
 * Point center is in screen coordinates. This is where mouse pointer is on the screen.
 * The layout is : GtkWindow has ClutterStage which contains ClutterActor scaleLayer
 * (in this function named self), which contains those blue
 * circles you see on animated gif.
 *
 * ScaleLayer is simply a huge invisible plane which contains all the objects an user
 * interacts with. User can pan and zoom it as he whishes.
 */
void ScaleLayer::zoomOut (const Point &center)
{
        double scaleX, scaleY;
        // This gets our actual zoom factor.
        clutter_actor_get_scale (self, &scaleX, &scaleY);
        // We decrease the zoom factor since this is a zoomOut method.
        float newScale = scaleX / 1.1;

        float cx1, cy1;
        // Like I said 'center' is in screen coords, so we convert it to scaleLayer coords
        clutter_actor_transform_stage_point (self, center.x, center.y, &cx1, &cy1);

        float scaleLayerNW, scaleLayerNH;
        clutter_actor_get_size (self, &scaleLayerNW, &scaleLayerNH);
        // We set pivot_point
        clutter_actor_set_pivot_point (self, double(cx1) / scaleLayerNW, double(cy1) / scaleLayerNH);
        // And finalyy perform the scalling. Fair enough, isn't it?
        clutter_actor_set_scale (self, newScale, newScale);
}
Here is the outcome of the above:

[caption id="" align="alignnone" width="640"]
![](http://iwasz.pl/files/zoomfail.gif) Zoom fail[/caption]

It is fine until you move the mouse cursor which changes the pivot point (center of the scale transformation) while scale is not == 1.0. I dunno why this happens. Apparently I do not understand affine transformations as well as I thought, or there is a bug in libclutter (I doubt it). The solution is to convert the pivot point from screen to scaleLayer coordinates before scaling (as I did), and again after scaling. The difference is in scaleLayer coordinates, so it must be converted back to screen coordinates, and the result can be used to cancel this offset you see on the second gif. Here is my current implementation:

void ScaleLayer::zoomIn (const Point &center)
{
        double x, y;
        clutter_actor_get_scale (self, &x, &y);

        if (x >= 10) {
                return;
        }

        double newScale = x * 1.1;

        if (newScale >= 10) {
                newScale = 10;
        }

        scale (center, newScale);
}

/*****************************************************************************/

void ScaleLayer::zoomOut (const Point &center)
{
        ClutterActor *stage = clutter_actor_get_parent (self);

        float stageW, stageH;
        clutter_actor_get_size (stage, &stageW, &stageH);

        float dim = std::max (stageW, stageH);
        double minScale = dim / SCALE_SURFACE_SIZE + 0.05;

        double scaleX, scaleY;
        clutter_actor_get_scale (self, &scaleX, &scaleY);

        if (scaleX <= minScale) { return; } scale (center, scaleX / 1.1); 
} 

/*****************************************************************************/ 

void ScaleLayer::scale (Point const &c, float newScale) { 
   Point center = c; float cx1, cy1; if (center == Point ()) { if (impl->lastCenter == Point ()) {
                        float stageW, stageH;
                        ClutterActor *stage = clutter_actor_get_parent (self);
                        clutter_actor_get_size (stage, &stageW, &stageH);
                        impl->lastCenter = Point (stageW / 2.0, stageH / 2.0);
                }

                center = impl->lastCenter;
        }
        else {
                impl->lastCenter = center;
        }

        clutter_actor_transform_stage_point (self, center.x, center.y, &cx1, &cy1);
        float scaleLayerNW, scaleLayerNH;
        clutter_actor_get_size (self, &scaleLayerNW, &scaleLayerNH);
        clutter_actor_set_pivot_point (self, double(cx1) / scaleLayerNW, double(cy1) / scaleLayerNH);
        clutter_actor_set_scale (self, newScale, newScale);

        // Idea taken from here : https://community.oracle.com/thread/1263955
        float cx2, cy2;
        clutter_actor_transform_stage_point (self, center.x, center.y, &cx2, &cy2);

        ClutterVertex vi1 = { 0, 0, 0 };
        ClutterVertex vo1;
        clutter_actor_apply_transform_to_point (self, &vi1, &vo1);

        ClutterVertex vi2 = { cx2 - cx1, cy2 - cy1, 0 };
        ClutterVertex vo2;
        clutter_actor_apply_transform_to_point (self, &vi2, &vo2);

        float mx = vo2.x - vo1.x;
        float my = vo2.y - vo1.y;

        clutter_actor_move_by (self, mx, my);
}
The whole project is here : 
[https://github.com/iwasz/data-flow-gui](https://github.com/iwasz/data-flow-gui)
Here's the thread which pointed me in right direction : 
[https://community.oracle.com/thread/1263955](https://community.oracle.com/thread/1263955)