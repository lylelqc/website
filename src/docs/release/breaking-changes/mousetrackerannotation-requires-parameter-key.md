---
title: MouseTrackerAnnotation requires parameter "key"
description: [Brief description similar to the "context" section below.]
---

# MouseTrackerAnnotation requires parameter "key"


## Context

Mouse events, such as when a mouse pointer has entered a region, exited, or
is hovering a region, are detected with the help of
`MouseTrackerAnnotation`s that are placed on interested regions during the
render phase. Upon each new frame or new event, `MouseTracker` compares the
annotations hovered by the mouse pointer before and after the change, then
dispatches callbacks accordingly. `MouseTracker` also tracks which
annotations are on the screen by requring the annotation's owners to attach
or detach the annotation promptly.

However, since the `MouseTrackerAnnotation` is a constant class, its properties
can't be changed without replacing the annotation as a whole, which triggers
the exit and enter callback if it's hovered by a mouse pointer.

This issue is a blocker to the plan of supporting mouse cursor, since it’s
very likely that the cursor assigned to a region needs to be changed.


## Description of change

Class `MouseTrackerAnnotation` now requires a new parameter `key`:

```diff
 class MouseTrackerAnnotation extends Diagnosticable {
   const MouseTrackerAnnotation({
     this.onEnter,
     this.onHover,
     this.onExit,
-  });
+    @required this.key,
+  }) : assert(key != null);

+  final LocalKey key;

   ...
 }
```

Three of class `MouseTracker`'s methods, `attachAnnotation`,
`detachAnnotation`, and `isAnnotationAttached`, instead of taking a
single parameter of `MouseTrackerAnnotation`, will now take a
single parameter `LocalKey`:

```diff
 class MouseTracker extends ChangeNotifier {
   ...

   @visibleForTesting
-  bool isAnnotationAttached(MouseTrackerAnnotation annotation);
+  bool isAnnotationAttached(LocalKey annotationKey);

-  void attachAnnotation(MouseTrackerAnnotation annotation);
+  void attachAnnotation(LocalKey annotationKey);

-  void detachAnnotation(MouseTrackerAnnotation annotation);
+  void detachAnnotation(LocalKey annotationKey);
 }
```

## Migration guide

There are 2 ways of migrating affected code. The first way requires little
work, while the second is much simpler when the annotation's properties
might change.

### The straightforward way
The most straightforward way is by providing
`MouseTrackerAnnotation`'s constructor with `key: UniqueKey()`, which is
sufficient because the annotation only needs to be equal to itself, since the
annotation was not designed to be mutable. Then change the calls to the
affected methods to using the annotation's key. For example:

```diff
 class MyRenderObject extends RenderObject {
   final MouseHoverAnnotation _hoverAnnotation = MouseTrackerAnnotation(
+    key: UniqueKey(),
     onEnter: _handleEnter,
     onExit: _handleExit,
   );

   @override
   void attach(PipelineOwner owner) {
     super.attach(owner);
-    RendererBinding.instance.mouseTracker.attachAnnotation(_hoverAnnotation);
+    RendererBinding.instance.mouseTracker.attachAnnotation(_hoverAnnotation.key);
   }

   @override
   void detach() {
-    RendererBinding.instance.mouseTracker.detachAnnotation(_hoverAnnotation);
+    RendererBinding.instance.mouseTracker.detachAnnotation(_hoverAnnotation.key);
     super.detach();
   }

   @override
   void paint(PaintingContext context, Offset offset) {
     final Layer layer = AnnotatedRegionLayer<MouseTrackerAnnotation>(
       _hoverAnnotation,
       size: size,
       offset: offset,
       opaque: opaque,
     );
     context.pushLayer(layer, super.paint, offset);
   }
 }
```

### The key-based way

Another way is to think that the annotation is now completely based on the
key: store the key instead of the annotation, and generate the annotation on
the fly. For example,

```diff
 class MyRenderObject extends RenderObject {
-  final MouseHoverAnnotation _hoverAnnotation = MouseTrackerAnnotation(
-    onEnter: _handleEnter,
-    onExit: _handleExit,
-  );
+  final LocalKey _hoverAnnotationKey = UniqueKey();

   @override
   void attach(PipelineOwner owner) {
     super.attach(owner);
-    RendererBinding.instance.mouseTracker.attachAnnotation(_hoverAnnotation);
+    RendererBinding.instance.mouseTracker.attachAnnotation(_hoverAnnotationKey);
   }

   @override
   void detach() {
-    RendererBinding.instance.mouseTracker.detachAnnotation(_hoverAnnotation);
+    RendererBinding.instance.mouseTracker.detachAnnotation(_hoverAnnotationKey);
     super.detach();
   }

   @override
   void paint(PaintingContext context, Offset offset) {
     final Layer layer = AnnotatedRegionLayer<MouseTrackerAnnotation>(
-      _hoverAnnotation,
+      MouseTrackerAnnotation(
+        onEnter: _handleEnter,
+        onExit: _handleExit,
+        key: _annotationKey,
+      ),
       size: size,
       offset: offset,
       opaque: opaque,
     );
     context.pushLayer(layer, super.paint, offset);
   }
 }
```

## Timeline

This change was made at v1.13.4.

## References

API documentation:
* [MouseTracker class](https://api.flutter.dev/flutter/gestures/MouseTracker-class.html)
* [MouseTrackerAnnotation class](https://api.flutter.dev/flutter/gestures/MouseTrackerAnnotation-class.html)

Relevant issues:
* [Add the ability to programmatically change pointer cursors](https://github.com/flutter/flutter/issues/31952)

Relevant PRs:
* [Add key to mouse annotations](https://github.com/flutter/flutter/pull/46034)